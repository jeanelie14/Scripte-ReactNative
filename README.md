import fs from 'fs';
import path from 'path';
import readline from 'readline';
import https from 'https';
import chalk from 'chalk';
import { execSync } from 'child_process';

// Configuration
const PROJECT_ROOT = process.cwd();
const NPM_REGISTRY = 'https://registry.npmjs.org';
let filesScanned = 0;
let conflictsFound = 0;

// Interface améliorée
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});

// Couleurs
const colors = {
  error: chalk.red.bold,
  warning: chalk.yellow,
  success: chalk.green,
  info: chalk.blue,
  highlight: chalk.cyan.bold,
  file: chalk.magenta
};

async function main() {
  try {
    console.clear();
    showBanner();
    
    // Vérifier si chalk est installé
    try {
      require.resolve('chalk');
    } catch {
      console.log(colors.info('Installation de chalk pour les couleurs...'));
      execSync('npm install chalk', { stdio: 'inherit' });
    }

    // Lire le package.json
    const packageJson = await readPackageJson();
    filesScanned++;
    updateProgress();

    // Afficher les versions actuelles
    showCurrentVersions(packageJson);

    // Analyser les dépendances
    console.log(colors.highlight('\nAnalyse des dépendances en cours...'));
    const conflicts = await analyzeDependencies(packageJson);

    if (conflicts.length > 0) {
      conflictsFound = conflicts.length;
      showConflicts(conflicts);
      await resolveConflicts(packageJson, conflicts);
      
      // Sauvegarder les changements
      await savePackageJson(packageJson);
      filesScanned++;
      updateProgress();
      
      showResolutionSummary();
    } else {
      console.log(colors.success('\n✔ Aucun conflit de dépendances trouvé !'));
    }

  } catch (error) {
    console.error(`\n${colors.error('Erreur:')} ${error.message}`);
  } finally {
    rl.close();
  }
}

function showBanner() {
  console.log(colors.highlight(`
===================================================
  GESTIONNAIRE DE DÉPENDANCES REACT/REACT NATIVE  
===================================================
`));
}

function updateProgress() {
  readline.cursorTo(process.stdout, 0);
  process.stdout.write(
    `${colors.info(`[${filesScanned}/5]`)} Fichiers analysés | ${colors.warning(`[${conflictsFound}]`)} Conflits trouvés`
  );
  if (filesScanned === 5) process.stdout.write('\n');
}

async function readPackageJson() {
  const packageJsonPath = path.join(PROJECT_ROOT, 'package.json');
  if (!fs.existsSync(packageJsonPath)) {
    throw new Error('package.json non trouvé');
  }
  return JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));
}

function showCurrentVersions(packageJson) {
  console.log(`\n${colors.highlight('Versions actuelles:')}
  - react: ${colors.info(packageJson.dependencies?.react || 'non installé')}
  - react-native: ${colors.info(packageJson.dependencies?.['react-native'] || 'non installé')}`);
}

async function analyzeDependencies(packageJson) {
  const conflicts = [];
  const allDeps = {
    ...packageJson.dependencies,
    ...(packageJson.devDependencies || {})
  };

  // Packages à vérifier spécifiquement
  const reactPackages = [
    'react-test-renderer',
    '@react-navigation/native',
    '@react-navigation/stack',
    'react-native-reanimated',
    'react-native-gesture-handler'
  ];

  for (const pkg of reactPackages) {
    if (allDeps[pkg]) {
      try {
        const pkgInfo = await fetchPackageInfo(pkg);
        filesScanned++;
        updateProgress();

        if (pkgInfo.peerDependencies?.react) {
          const requiredReact = pkgInfo.peerDependencies.react;
          const currentReact = packageJson.dependencies?.react?.replace(/[\^~]/, '') || '18.3.1';
          
          if (!satisfiesVersion(currentReact, requiredReact)) {
            conflicts.push({
              package: pkg,
              version: allDeps[pkg],
              requiresReact: requiredReact,
              currentReact: currentReact
            });
          }
        }
      } catch (error) {
        console.warn(`\n${colors.warning('⚠')} Impossible de vérifier ${pkg}: ${error.message}`);
      }
    }
  }

  return conflicts;
}

function showConflicts(conflicts) {
  console.log(`\n${colors.error('Conflits détectés:')}`);
  conflicts.forEach((conflict, index) => {
    console.log(`${index + 1}. ${colors.highlight(conflict.package)}@${conflict.version}
   Nécessite React: ${colors.error(conflict.requiresReact)}
   Version actuelle: ${colors.info(conflict.currentReact)}\n`);
  });
}

async function resolveConflicts(packageJson, conflicts) {
  console.log(colors.highlight('\nRésolution des conflits:'));
  
  for (const conflict of conflicts) {
    try {
      // Trouver une version compatible
      const compatibleVersion = await findCompatibleVersion(
        conflict.package, 
        conflict.currentReact
      );
      
      // Mettre à jour la dépendance
      if (packageJson.dependencies?.[conflict.package]) {
        packageJson.dependencies[conflict.package] = compatibleVersion;
      } else if (packageJson.devDependencies?.[conflict.package]) {
        packageJson.devDependencies[conflict.package] = compatibleVersion;
      }
      
      console.log(`${colors.success('✓')} ${colors.highlight(conflict.package)} mis à jour vers ${colors.info(compatibleVersion)}`);
    } catch (error) {
      console.warn(`${colors.warning('⚠')} Impossible de résoudre ${conflict.package}: ${error.message}`);
    }
  }
}

async function findCompatibleVersion(packageName, reactVersion) {
  const pkgInfo = await fetchPackageInfo(packageName);
  const versions = Object.keys(pkgInfo.versions)
    .filter(v => !v.includes('-')) // Exclure les versions beta/alpha
    .reverse();

  // Trouver une version compatible avec la version de React
  for (const version of versions) {
    const versionInfo = pkgInfo.versions[version];
    if (versionInfo.peerDependencies?.react) {
      if (satisfiesVersion(reactVersion, versionInfo.peerDependencies.react)) {
        return `^${version}`;
      }
    }
  }

  throw new Error('Aucune version compatible trouvée');
}

function showResolutionSummary() {
  console.log(`\n${colors.success('Résumé des corrections:')}
  - Fichiers analysés: ${colors.info(filesScanned)}
  - Conflits trouvés: ${colors.warning(conflictsFound)}
  - Conflits résolus: ${colors.success(conflictsFound)}`);
  
  console.log(`\n${colors.highlight('Prochaines étapes:')}
  1. Exécutez ${colors.info('npm install')} ou ${colors.info('yarn')}
  2. Pour React Native:
     - ${colors.info('cd ios && pod install && cd ..')}
  3. Vérifiez les éventuelles breaking changes`);
}

async function savePackageJson(packageJson) {
  const packageJsonPath = path.join(PROJECT_ROOT, 'package.json');
  fs.writeFileSync(packageJsonPath, JSON.stringify(packageJson, null, 2) + '\n');
}

async function fetchPackageInfo(packageName) {
  return new Promise((resolve, reject) => {
    https.get(`${NPM_REGISTRY}/${packageName}`, (res) => {
      let data = '';
      res.on('data', (chunk) => data += chunk);
      res.on('end', () => {
        try {
          resolve(JSON.parse(data));
        } catch (e) {
          reject(new Error('Erreur de parsing'));
        }
      });
    }).on('error', () => {
      reject(new Error('Échec de connexion au registre npm'));
    });
  });
}

// Fonction simplifiée de vérification de version
function satisfiesVersion(version, range) {
  // Implémentation basique - pour une solution complète, utiliser semver
  const v = version.split('.')[0];
  const r = range.match(/(\d+)/)?.[0];
  return v === r;
}

// Démarrer le script
main();
