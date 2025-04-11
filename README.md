import fs from 'fs';
import path from 'path';
import readline from 'readline';
import https from 'https';
import {execSync} from 'child_process';

// Configuration
const PROJECT_ROOT = process.cwd();
const NPM_REGISTRY = 'https://registry.npmjs.org';
let filesScanned = 0;

// Interface améliorée
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

// Fonction pour poser une question
function askQuestion(question) {
  return new Promise(resolve => {
    rl.question(question, answer => {
      resolve(answer.trim());
    });
  });
}

// Animation de progression
function showProgress(current, total, message) {
  readline.cursorTo(process.stdout, 0);
  process.stdout.write(`[${current}/${total}] ${message}`);
  if (current === total) {
    process.stdout.write('\n');
  }
}

async function main() {
  try {
    console.log('\n=== Analyse des dépendances React/React Native ===\n');

    // Lire le package.json
    const packageJson = await readPackageJson();
    filesScanned++;
    showProgress(filesScanned, 5, 'package.json analysé');

    // Afficher les versions actuelles
    console.log(`\nVersions actuelles:
  - react: ${packageJson.dependencies?.react || 'non installé'}
  - react-native: ${
    packageJson.dependencies?.['react-native'] || 'non installé'
  }`);

    // Détecter les conflits
    console.log('\nDétection des conflits...');
    await detectConflicts(packageJson);

    // Résolution des problèmes
    console.log('\nRésolution des problèmes:');
    await resolvePeerDependencies(packageJson);

    // Sauvegarder les changements
    await savePackageJson(packageJson);
    filesScanned++;
    showProgress(filesScanned, 5, 'package.json mis à jour');

    console.log('\n=== Opération terminée avec succès ===');
    console.log('Exécutez "npm install" pour appliquer les changements');
  } catch (error) {
    console.error(`\nErreur: ${error.message}`);
  } finally {
    rl.close();
  }
}

async function readPackageJson() {
  const packageJsonPath = path.join(PROJECT_ROOT, 'package.json');
  if (!fs.existsSync(packageJsonPath)) {
    throw new Error('package.json non trouvé');
  }
  return JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));
}

async function savePackageJson(packageJson) {
  const packageJsonPath = path.join(PROJECT_ROOT, 'package.json');
  fs.writeFileSync(
    packageJsonPath,
    JSON.stringify(packageJson, null, 2) + '\n',
  );
}

async function detectConflicts(packageJson) {
  const conflicts = [];
  const deps = {...packageJson.dependencies, ...packageJson.devDependencies};

  for (const [dep, version] of Object.entries(deps)) {
    try {
      const pkgInfo = await fetchPackageInfo(dep);
      filesScanned++;
      showProgress(filesScanned, 5, `Analyse de ${dep}`);

      if (pkgInfo.peerDependencies?.react) {
        const cleanVersion = version.replace(/[\^~]/, '');
        const peerReact = pkgInfo.peerDependencies.react;

        if (!satisfiesVersion(cleanVersion, peerReact)) {
          conflicts.push({
            package: dep,
            version: version,
            requiresReact: peerReact,
            currentReact: packageJson.dependencies?.react,
          });
        }
      }
    } catch (error) {
      console.warn(`⚠ Impossible de vérifier ${dep}: ${error.message}`);
    }
  }

  if (conflicts.length > 0) {
    console.log('\nConflits détectés:');
    conflicts.forEach(conflict => {
      console.log(
        `- ${conflict.package}@${conflict.version} nécessite React ${conflict.requiresReact}`,
      );
    });
  } else {
    console.log('\nAucun conflit majeur détecté');
  }

  return conflicts;
}

async function resolvePeerDependencies(packageJson) {
  // Cas spécifique de react-test-renderer
  if (packageJson.devDependencies?.['react-test-renderer']) {
    const currentReact =
      packageJson.dependencies?.react?.replace(/[\^~]/, '') || '18.3.1';
    const reactMajor = currentReact.split('.')[0];

    try {
      const compatibleVersion = await findCompatibleVersion(
        'react-test-renderer',
        `${reactMajor}.x`,
      );

      if (compatibleVersion) {
        packageJson.devDependencies[
          'react-test-renderer'
        ] = `^${compatibleVersion}`;
        console.log(
          `✓ react-test-renderer mis à jour vers ${compatibleVersion} pour compatibilité`,
        );
      }
    } catch (error) {
      console.warn(
        `⚠ Impossible de mettre à jour react-test-renderer: ${error.message}`,
      );
    }
  }
}

async function findCompatibleVersion(packageName, reactRange) {
  const pkgInfo = await fetchPackageInfo(packageName);
  const versions = Object.keys(pkgInfo.versions)
    .filter(v => !v.includes('-')) // Exclure les versions beta/alpha
    .reverse();

  // Trouver une version compatible avec la plage React
  for (const version of versions) {
    const versionInfo = pkgInfo.versions[version];
    if (versionInfo.peerDependencies?.react) {
      if (satisfiesVersion(reactRange, versionInfo.peerDependencies.react)) {
        return version;
      }
    }
  }

  throw new Error('Aucune version compatible trouvée');
}

async function fetchPackageInfo(packageName) {
  return new Promise((resolve, reject) => {
    https
      .get(`${NPM_REGISTRY}/${packageName}`, res => {
        let data = '';
        res.on('data', chunk => (data += chunk));
        res.on('end', () => {
          try {
            resolve(JSON.parse(data));
          } catch (e) {
            reject(new Error('Erreur de parsing'));
          }
        });
      })
      .on('error', () => {
        reject(new Error('Échec de connexion au registre npm'));
      });
  });
}

// Fonction simplifiée de vérification de version
function satisfiesVersion(version, range) {
  const v = version.split('.')[0];
  const r = range.match(/[\d]+/)?.[0];
  return v === r;
}

// Démarrer le script
main();
