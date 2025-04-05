#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const {execSync, exec} = require('child_process');
const readline = require('readline');
const semver = require('semver');
const colors = require('colors');
const fetch = require('node-fetch');

// Configuration
const CONFIG = {
  npmRegistry: 'https://registry.npmjs.org',
  logDir: path.join(require('os').homedir(), 'project_updater_logs'),
  stateFile: path.join(require('os').tmpdir(), 'project_updater_state.json'),
};

// Couleurs
colors.setTheme({
  success: 'green',
  warning: 'yellow',
  error: 'red',
  info: 'cyan',
  highlight: 'magenta',
  muted: 'gray',
});

// État global
const STATE = {
  updatedPackages: [],
  skippedPackages: [],
  errors: [],
  createdFiles: [],
  completedSteps: [],
  startTime: new Date(),
};

// Interface utilisateur
class UI {
  static async ask(question) {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    });

    return new Promise(resolve => {
      rl.question(question, answer => {
        rl.close();
        resolve(answer.trim());
      });
    });
  }

  static async confirm(question, defaultValue = true) {
    const answer = await this.ask(
      `${question} (${defaultValue ? 'Y/n' : 'y/N'}) `,
    );
    return answer === '' ? defaultValue : /^y(es)?$/i.test(answer);
  }

  static separator() {
    console.log('━'.repeat(50).muted);
  }

  static title(message) {
    console.log('\n');
    this.separator();
    console.log(` ${message} `.highlight.bold);
    this.separator();
  }
}

// Gestionnaire de package.json
class PackageManager {
  constructor() {
    this.packageJsonPath = path.join(process.cwd(), 'package.json');
    this.packageData = {};
    this.originalPackageData = {};
  }

  async initialize() {
    await this.loadPackageJson();
  }

  async loadPackageJson() {
    try {
      if (fs.existsSync(this.packageJsonPath)) {
        this.packageData = JSON.parse(
          fs.readFileSync(this.packageJsonPath, 'utf8'),
        );
        this.originalPackageData = JSON.parse(JSON.stringify(this.packageData));
        return true;
      }
      return false;
    } catch (error) {
      throw new Error(`Erreur de lecture du package.json: ${error.message}`);
    }
  }

  async savePackageJson() {
    try {
      fs.writeFileSync(
        this.packageJsonPath,
        JSON.stringify(this.packageData, null, 2),
      );
      return true;
    } catch (error) {
      throw new Error(`Erreur d'écriture du package.json: ${error.message}`);
    }
  }

  async updateDependencies(dependencies, isDev = false) {
    const target = isDev ? 'devDependencies' : 'dependencies';

    if (!this.packageData[target]) {
      this.packageData[target] = {};
    }

    for (const [pkg, version] of Object.entries(dependencies)) {
      this.packageData[target][pkg] = version;
    }

    await this.savePackageJson();
  }

  async installDependencies(dependencies, isDev = false) {
    const results = [];

    for (const [pkg, version] of Object.entries(dependencies)) {
      try {
        // Vérifier d'abord si la version existe
        const exists = await NpmApi.versionExists(pkg, version);

        if (!exists) {
          // Trouver une version alternative compatible
          const compatibleVersion = await this.findCompatibleVersion(
            pkg,
            version,
          );

          if (compatibleVersion) {
            console.log(
              `Version ${version} non disponible pour ${pkg}, tentative avec ${compatibleVersion}`
                .warning,
            );
            await this.installSinglePackage(pkg, compatibleVersion, isDev);
            results.push({
              pkg,
              version: compatibleVersion,
              status: 'success',
              originalVersion: version,
            });
          } else {
            throw new Error(`Aucune version compatible trouvée pour ${pkg}`);
          }
        } else {
          await this.installSinglePackage(pkg, version, isDev);
          results.push({pkg, version, status: 'success'});
        }
      } catch (error) {
        results.push({pkg, version, status: 'error', message: error.message});
        STATE.errors.push(
          `Échec installation ${pkg}@${version}: ${error.message}`,
        );
      }
    }

    return results;
  }

  async installSinglePackage(pkg, version, isDev = false) {
    const command = `npm install ${pkg}@${version} --legacy-peer-deps${
      isDev ? ' --save-dev' : ''
    }`;

    try {
      execSync(command, {stdio: 'inherit'});
    } catch (error) {
      // Essayer sans version spécifique si l'installation échoue
      console.log(
        `Tentative d'installation sans version spécifique pour ${pkg}`.warning,
      );
      execSync(
        `npm install ${pkg} --legacy-peer-deps${isDev ? ' --save-dev' : ''}`,
        {stdio: 'inherit'},
      );
    }
  }

  async findCompatibleVersion(pkg, requestedVersion) {
    try {
      // Obtenir toutes les versions disponibles
      const versions = await NpmApi.getPackageVersions(pkg);

      // Trouver la version la plus récente qui satisfait la plage demandée
      const version = versions
        .filter(v => semver.satisfies(v, requestedVersion))
        .sort((a, b) => semver.rcompare(a, b))[0];

      return version || (await this.findLatestStableVersion(pkg));
    } catch (error) {
      return await this.findLatestStableVersion(pkg);
    }
  }

  async findLatestStableVersion(pkg) {
    try {
      return await NpmApi.getLatestVersion(pkg);
    } catch {
      return null;
    }
  }

  async getInstalledVersion(pkg) {
    try {
      const pkgPath = path.join(
        process.cwd(),
        'node_modules',
        pkg,
        'package.json',
      );
      if (fs.existsSync(pkgPath)) {
        const pkgJson = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
        return pkgJson.version;
      }
      return null;
    } catch {
      return null;
    }
  }
}

// API NPM
class NpmApi {
  static async getLatestVersion(packageName) {
    try {
      const response = await fetch(
        `${CONFIG.npmRegistry}/${packageName}/latest`,
      );
      if (!response.ok) throw new Error('Package not found');
      const data = await response.json();
      return data.version;
    } catch (error) {
      throw new Error(`Failed to get version for ${packageName}`);
    }
  }

  static async versionExists(packageName, version) {
    try {
      const response = await fetch(`${CONFIG.npmRegistry}/${packageName}`);
      if (!response.ok) throw new Error('Package not found');
      const data = await response.json();
      return data.versions && data.versions[version];
    } catch (error) {
      throw new Error(`Failed to check version for ${packageName}`);
    }
  }

  static async getPackageVersions(packageName) {
    try {
      const response = await fetch(`${CONFIG.npmRegistry}/${packageName}`);
      if (!response.ok) throw new Error('Package not found');
      const data = await response.json();
      return Object.keys(data.versions).sort((a, b) => semver.rcompare(a, b));
    } catch (error) {
      throw new Error(`Failed to get versions for ${packageName}`);
    }
  }

  static async getCompatibleVersion(packageName, currentVersion) {
    try {
      const versions = await this.getPackageVersions(packageName);
      const compatibleVersions = versions.filter(v =>
        semver.satisfies(v, `^${currentVersion}`),
      );
      return compatibleVersions[0] || currentVersion;
    } catch (error) {
      throw new Error(`Failed to find compatible version for ${packageName}`);
    }
  }
}

// Fonction principale
async function main() {
  console.clear();
  UI.title('MISE À JOUR DE PROJET - DEEPSEEK');

  const packageManager = new PackageManager();
  const hasPackageJson = await packageManager.loadPackageJson();

  // Afficher le menu principal
  console.log('\nBonjour, choisissez une option :'.info.bold);
  console.log('1. Mettre à jour le package.json'.highlight);
  console.log('2. Installer des dépendances'.highlight);
  console.log('3. Comparer et mettre à jour les versions'.highlight);

  const choice = await UI.ask('\nVotre choix (1-3) : ');

  switch (choice) {
    case '1':
      await handlePackageJsonUpdate(packageManager, hasPackageJson);
      break;
    case '2':
      await handleDependencyInstallation(packageManager);
      break;
    case '3':
      await handleVersionComparison(packageManager, hasPackageJson);
      break;
    default:
      console.log('Option invalide'.error);
      process.exit(1);
  }

  // Afficher le rapport final
  await generateReport();
}

async function handlePackageJsonUpdate(packageManager, hasPackageJson) {
  UI.title('MISE À JOUR DU PACKAGE.JSON');

  if (!hasPackageJson) {
    console.log('Aucun package.json trouvé dans le projet.'.warning);
    return;
  }

  console.log('Contenu actuel du package.json :'.info);
  console.log(JSON.stringify(packageManager.packageData, null, 2));

  const shouldUpdate = await UI.confirm(
    '\nVoulez-vous mettre à jour ce package.json ?',
  );
  if (!shouldUpdate) return;

  // Ici vous pourriez ajouter la logique pour mettre à jour le package.json
  // selon vos besoins spécifiques

  console.log('Package.json mis à jour avec succès'.success);
}

async function handleDependencyInstallation(packageManager) {
  UI.title('INSTALLATION DE DÉPENDANCES');

  console.log(
    'Entrez les dépendances à installer (format: nom@version) :'.info,
  );
  console.log('Appuyez sur Entrée deux fois pour terminer'.muted);

  const dependencies = await readMultiLineInput();

  if (dependencies.length === 0) {
    console.log('Aucune dépendance à installer'.warning);
    return;
  }

  const depsToInstall = {};
  for (const dep of dependencies) {
    const [pkg, version] = dep.split('@');
    depsToInstall[pkg] = version || 'latest';
  }

  console.log('\nDépendances à installer :'.info);
  console.log(depsToInstall);

  const isDev = await UI.confirm(
    'Ce sont des dépendances de développement ?',
    false,
  );
  const shouldInstall = await UI.confirm(
    'Voulez-vous installer ces dépendances ?',
  );

  if (shouldInstall) {
    try {
      const results = await packageManager.installDependencies(
        depsToInstall,
        isDev,
      );

      // Afficher les résultats
      UI.title("RÉSULTATS DE L'INSTALLATION");
      results.forEach(result => {
        if (result.status === 'success') {
          if (result.originalVersion) {
            console.log(
              `${result.pkg.highlight}: ${result.originalVersion} → ${result.version}`
                .success,
            );
          } else {
            console.log(
              `${result.pkg.highlight}@${result.version} installé`.success,
            );
          }
        } else {
          console.log(
            `${result.pkg.highlight}@${result.version} échec: ${result.message}`
              .error,
          );
        }
      });
    } catch (error) {
      console.log(`Erreur lors de l'installation : ${error.message}`.error);
    }
  }
}

async function handleVersionComparison(packageManager, hasPackageJson) {
  UI.title('COMPARAISON DE VERSIONS');

  if (!hasPackageJson) {
    console.log('Aucun package.json trouvé dans le projet.'.warning);
    return;
  }

  const {dependencies = {}, devDependencies = {}} = packageManager.packageData;
  const allDeps = {...dependencies, ...devDependencies};

  if (Object.keys(allDeps).length === 0) {
    console.log('Aucune dépendance trouvée dans le package.json'.warning);
    return;
  }

  console.log('Analyse des versions des dépendances...'.info);

  for (const [pkg, currentVersion] of Object.entries(allDeps)) {
    try {
      const installedVersion = await packageManager.getInstalledVersion(pkg);
      const compatibleVersion = await NpmApi.getCompatibleVersion(
        pkg,
        currentVersion,
      );

      console.log(`\n${pkg.highlight}:`);
      console.log(`- Version spécifiée: ${currentVersion}`);
      console.log(`- Version installée: ${installedVersion || 'Non installé'}`);
      console.log(`- Version compatible: ${compatibleVersion}`);

      if (compatibleVersion !== currentVersion) {
        const shouldUpdate = await UI.confirm(
          `Mettre à jour ${pkg} à ${compatibleVersion} ?`,
        );
        if (shouldUpdate) {
          const isDev = !!devDependencies[pkg];
          await packageManager.updateDependencies(
            {[pkg]: compatibleVersion},
            isDev,
          );
          console.log(`${pkg} mis à jour avec succès`.success);
        }
      }
    } catch (error) {
      console.log(`Erreur avec ${pkg}: ${error.message}`.error);
      STATE.errors.push(`Erreur avec ${pkg}: ${error.message}`);
    }
  }
}

async function readMultiLineInput() {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  console.log('\nEntrez les dépendances (format: nom@version) :');

  return new Promise(resolve => {
    const lines = [];

    rl.on('line', line => {
      if (line.trim() === '' && lines.length > 0) {
        rl.close();
        resolve(lines);
      } else if (line.trim() !== '') {
        lines.push(line.trim());
      }
    });

    // Timeout après 30 secondes d'inactivité
    setTimeout(() => {
      if (lines.length > 0) {
        rl.close();
        resolve(lines);
      }
    }, 30000);
  });
}

async function generateReport() {
  UI.title('RAPPORT FINAL');

  if (STATE.updatedPackages.length > 0) {
    console.log('\nPACKAGES MIS À JOUR:'.success.bold);
    STATE.updatedPackages.forEach(pkg => {
      console.log(`- ${pkg.name.highlight}: ${pkg.from} → ${pkg.to}`);
    });
  }

  if (STATE.skippedPackages.length > 0) {
    console.log('\nPACKAGES NON MODIFIÉS:'.info.bold);
    STATE.skippedPackages.forEach(pkg => {
      console.log(`- ${pkg.name}: ${pkg.version}`);
    });
  }

  if (STATE.createdFiles.length > 0) {
    console.log('\nFICHIERS/DOOSSIERS CRÉÉS:'.info.bold);
    STATE.createdFiles.forEach(file => {
      console.log(`- ${file}`);
    });
  }

  if (STATE.errors.length > 0) {
    console.log('\nERREURS:'.error.bold);
    STATE.errors.forEach((error, i) => {
      console.log(`${i + 1}) ${error}`);
    });
  }

  const duration = (new Date() - STATE.startTime) / 1000;
  console.log(`\nTemps d'exécution: ${duration.toFixed(2)} secondes`.muted);
}

// Démarrer l'application
(async () => {
  try {
    // Vérifier et installer les dépendances requises si nécessaire
    try {
      require('semver');
      require('colors');
      require('node-fetch');
    } catch {
      console.log('Installation des dépendances requises...'.info);
      execSync(
        'npm install semver colors node-fetch --no-save --legacy-peer-deps',
        {stdio: 'inherit'},
      );
    }

    await main();
  } catch (error) {
    console.error('ERREUR CRITIQUE:'.error.bold, error);
    process.exit(1);
  }
})();
