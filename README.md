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
  gradleVersion: null,
  androidConfig: null,
  reactVersion: null,
  reactNativeVersion: null,
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

// Gestionnaire de structure de projet
class ProjectStructureManager {
  static async createStructure() {
    UI.title('STRUCTURATION DU PROJET');

    const hasPredefinedStructure = await UI.confirm(
      'Avez-vous une structure prédéfinie à créer ?',
    );
    if (!hasPredefinedStructure) return;

    console.log(
      '\nEntrez la structure de votre projet (un élément par ligne) :'.info,
    );
    console.log('Exemple:'.muted);
    console.log('src/'.muted);
    console.log('  components/'.muted);
    console.log('    Button.js'.muted);
    console.log('  assets/images/'.muted);
    console.log('\nAppuyez sur Entrée deux fois pour terminer'.muted);

    const structure = await readMultiLineInput();
    if (structure.length === 0) {
      console.log('Aucune structure à créer'.warning);
      return;
    }

    console.log('\nCréation de la structure...'.info);
    for (const item of structure) {
      const fullPath = path.join(process.cwd(), item.trim());
      try {
        if (item.includes('.')) {
          // C'est un fichier
          const dir = path.dirname(fullPath);
          if (!fs.existsSync(dir)) {
            fs.mkdirSync(dir, {recursive: true});
          }
          fs.writeFileSync(fullPath, '');
          STATE.createdFiles.push(`Fichier créé: ${item}`);
        } else {
          // C'est un dossier
          fs.mkdirSync(fullPath, {recursive: true});
          STATE.createdFiles.push(`Dossier créé: ${item}`);
        }
      } catch (error) {
        STATE.errors.push(`Erreur création ${item}: ${error.message}`);
      }
    }

    console.log('Structure créée avec succès'.success);
  }
}

// Gestionnaire d'environnement
class EnvironmentManager {
  static detectEnvironment(packageData) {
    if (packageData.dependencies) {
      STATE.reactVersion = packageData.dependencies.react;
      STATE.reactNativeVersion = packageData.dependencies['react-native'];
    }
    this.detectGradleVersion();
  }

  static detectGradleVersion() {
    try {
      const wrapperPropsPath = path.join(
        process.cwd(),
        'android',
        'gradle',
        'wrapper',
        'gradle-wrapper.properties',
      );
      if (fs.existsSync(wrapperPropsPath)) {
        const content = fs.readFileSync(wrapperPropsPath, 'utf8');
        const versionMatch = content.match(/gradle-(\d+\.\d+(\.\d+)?)-/);
        if (versionMatch) {
          STATE.gradleVersion = versionMatch[1];
        }
      }
    } catch (error) {
      console.log('Erreur lecture version Gradle'.error, error.message);
    }
  }

  static async checkDependencyCompatibility(pkg, version) {
    if (STATE.reactVersion && STATE.reactNativeVersion) {
      const reactCompatibility = await this.checkReactCompatibility(
        pkg,
        version,
      );
      if (!reactCompatibility.compatible) {
        return {
          compatible: false,
          message: `Incompatible avec React ${STATE.reactVersion} et React Native ${STATE.reactNativeVersion}`,
        };
      }
    }

    if (STATE.gradleVersion && this.isNativeDependency(pkg)) {
      const gradleCompatibility = await this.checkGradleCompatibility(
        pkg,
        version,
      );
      if (!gradleCompatibility.compatible) {
        return {
          compatible: false,
          message: `Incompatible avec Gradle ${STATE.gradleVersion}`,
        };
      }
    }

    return {compatible: true};
  }

  static async getCompatibleVersions(pkg, currentVersion) {
    try {
      const versions = await NpmApi.getPackageVersions(pkg);
      const reactRange = STATE.reactVersion
        ? `react@${STATE.reactVersion}`
        : '';
      const reactNativeRange = STATE.reactNativeVersion
        ? `react-native@${STATE.reactNativeVersion}`
        : '';

      let compatibleVersions = versions.filter(
        v =>
          semver.satisfies(v, currentVersion) &&
          this.checkPeerDependencies(pkg, v, reactRange, reactNativeRange),
      );

      if (compatibleVersions.length === 0) {
        compatibleVersions = versions.filter(v =>
          this.checkPeerDependencies(pkg, v, reactRange, reactNativeRange),
        );
      }

      return compatibleVersions
        .sort((a, b) => semver.rcompare(a, b))
        .slice(0, 3);
    } catch (error) {
      console.log(
        `Erreur recherche versions compatibles pour ${pkg}`.error,
        error.message,
      );
      return [];
    }
  }

  static async checkReactCompatibility(pkg, version) {
    try {
      const response = await fetch(`${CONFIG.npmRegistry}/${pkg}/${version}`);
      if (!response.ok) return {compatible: false};

      const pkgData = await response.json();
      const peerDeps = pkgData.peerDependencies || {};

      if (
        peerDeps.react &&
        !semver.satisfies(STATE.reactVersion, peerDeps.react)
      ) {
        return {
          compatible: false,
          message: `Nécessite React ${peerDeps.react} mais le projet utilise ${STATE.reactVersion}`,
        };
      }

      if (
        peerDeps['react-native'] &&
        !semver.satisfies(STATE.reactNativeVersion, peerDeps['react-native'])
      ) {
        return {
          compatible: false,
          message: `Nécessite React Native ${peerDeps['react-native']} mais le projet utilise ${STATE.reactNativeVersion}`,
        };
      }

      return {compatible: true};
    } catch {
      return {compatible: true};
    }
  }

  static async checkGradleCompatibility(pkg, version) {
    const gradleRequirements = {
      'react-native': '>=6.0',
      'react-native-reanimated': '>=6.0',
      'react-native-gesture-handler': '>=6.0',
      'react-native-screens': '>=6.0',
      'react-native-vector-icons': '>=6.0',
    };

    if (gradleRequirements[pkg]) {
      return {
        compatible: semver.satisfies(
          STATE.gradleVersion,
          gradleRequirements[pkg],
        ),
        message: `Nécessite Gradle ${gradleRequirements[pkg]} mais le projet utilise ${STATE.gradleVersion}`,
      };
    }

    return {compatible: true};
  }

  static async checkPeerDependencies(
    pkg,
    version,
    reactRange,
    reactNativeRange,
  ) {
    try {
      const response = await fetch(`${CONFIG.npmRegistry}/${pkg}/${version}`);
      if (!response.ok) return false;

      const pkgData = await response.json();
      const peerDeps = pkgData.peerDependencies || {};

      if (
        peerDeps.react &&
        reactRange &&
        !semver.satisfies(reactRange, peerDeps.react)
      ) {
        return false;
      }

      if (
        peerDeps['react-native'] &&
        reactNativeRange &&
        !semver.satisfies(reactNativeRange, peerDeps['react-native'])
      ) {
        return false;
      }

      return true;
    } catch {
      return true;
    }
  }

  static isNativeDependency(pkg) {
    const nativeDependencies = [
      'react-native',
      'react-native-reanimated',
      'react-native-gesture-handler',
      'react-native-screens',
      'react-native-vector-icons',
      'react-native-camera',
      'react-native-maps',
    ];

    return nativeDependencies.includes(pkg);
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
        const exists = await NpmApi.versionExists(pkg, version);

        if (!exists) {
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
      const versions = await NpmApi.getPackageVersions(pkg);
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

// Fonctions utilitaires
async function readMultiLineInput() {
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

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

// Fonctions principales
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

  console.log('Package.json mis à jour avec succès'.success);
}

async function handleDependencyInstallation(packageManager) {
  UI.title('INSTALLATION DE DÉPENDANCES');

  if (STATE.reactVersion) {
    console.log(`\nReact: ${STATE.reactVersion.highlight}`);
  }
  if (STATE.reactNativeVersion) {
    console.log(`React Native: ${STATE.reactNativeVersion.highlight}`);
  }
  if (STATE.gradleVersion) {
    console.log(`Gradle: ${STATE.gradleVersion.highlight}`);
  }

  console.log(
    '\nEntrez les dépendances à installer (format: nom@version) :'.info,
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

  // Vérifier la compatibilité
  const compatibilityIssues = [];
  for (const [pkg, version] of Object.entries(depsToInstall)) {
    const compatibility = await EnvironmentManager.checkDependencyCompatibility(
      pkg,
      version,
    );
    if (!compatibility.compatible) {
      const compatibleVersions = await EnvironmentManager.getCompatibleVersions(
        pkg,
        version,
      );
      compatibilityIssues.push({
        pkg,
        version,
        message: compatibility.message,
        compatibleVersions,
      });

      console.log(
        `\n⚠️ ${pkg.highlight}@${version}: ${compatibility.message}`.warning,
      );
      if (compatibleVersions.length > 0) {
        console.log(`Versions compatibles disponibles :`.info);
        compatibleVersions.forEach(v => console.log(`- ${v}`.highlight));
      } else {
        console.log(`Aucune version compatible trouvée`.error);
      }
    }
  }

  if (compatibilityIssues.length > 0) {
    const shouldContinue = await UI.confirm(
      '\nContinuer malgré les problèmes de compatibilité ?',
      false,
    );
    if (!shouldContinue) {
      const shouldUpdate = await UI.confirm(
        'Voulez-vous utiliser les versions compatibles suggérées ?',
      );
      if (shouldUpdate) {
        for (const issue of compatibilityIssues) {
          if (issue.compatibleVersions.length > 0) {
            depsToInstall[issue.pkg] = issue.compatibleVersions[0];
            console.log(
              `Utilisation de ${issue.pkg}@${issue.compatibleVersions[0]} à la place`
                .success,
            );
          }
        }
      } else {
        return;
      }
    }
  }

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

// Gestionnaire des fichiers Android
class AndroidFilesUpdater {
  static async updateAndroidFiles() {
    UI.title('MISE À JOUR DES FICHIERS ANDROID');

    if (!STATE.reactNativeVersion) {
      console.log('React Native non détecté dans le projet'.warning);
      return;
    }

    const androidDir = path.join(process.cwd(), 'android');
    if (!fs.existsSync(androidDir)) {
      console.log('Dossier Android non trouvé'.warning);
      return;
    }

    console.log(`Version React Native détectée: ${STATE.reactNativeVersion.highlight}`);

    try {
      const recommendedVersions = await this.fetchRecommendedVersions(
        STATE.reactNativeVersion,
      );

      console.log('\nVersions recommandées:'.info);
      console.log(`- Gradle: ${recommendedVersions.gradle}`);
      console.log(`- Kotlin: ${recommendedVersions.kotlin}`);
      console.log(`- Android Gradle Plugin: ${recommendedVersions.androidGradlePlugin}`);

      const shouldProceed = await UI.confirm(
        '\nVoulez-vous mettre à jour les fichiers Android ?',
        true
      );
      if (!shouldProceed) return;

      await this.updateGradleWrapper(androidDir, recommendedVersions.gradle);
      await this.updateProjectBuildGradle(
        androidDir,
        recommendedVersions.androidGradlePlugin,
        recommendedVersions.kotlin,
      );
      await this.updateGradleProperties(androidDir, recommendedVersions);

      console.log('\nMise à jour des fichiers Android terminée avec succès!'.success);
      
      const shouldRunClean = await UI.confirm(
        'Voulez-vous exécuter "gradlew clean" maintenant ?',
        true
      );
      
      if (shouldRunClean) {
        try {
          console.log('Exécution de gradlew clean...'.info);
          execSync('cd android && ./gradlew clean --no-daemon --stacktrace', {
            stdio: 'inherit',
            timeout: 600000
          });
          console.log('Nettoyage Gradle terminé avec succès'.success);
        } catch (error) {
          console.log('Erreur lors du nettoyage Gradle:'.error, error.message);
          STATE.errors.push(`Erreur gradlew clean: ${error.message}`);
        }
      } else {
        console.log(
          'Exécutez manuellement "cd android && ./gradlew clean" pour terminer la configuration'.info,
        );
      }
    } catch (error) {
      console.log(`Erreur lors de la mise à jour: ${error.message}`.error);
      STATE.errors.push(`Erreur mise à jour Android: ${error.message}`);
    }
  }

  static async fetchRecommendedVersions(rnVersion) {
    try {
      // Versions recommandées pour React Native 0.76+
      const recommendedVersions = {
        gradle: '8.3',
        kotlin: '1.9.20',
        androidGradlePlugin: '8.1.1'
      };

      // Fallback pour les versions plus anciennes
      const versionMap = {
        '0.72': {
          gradle: '7.5.1',
          kotlin: '1.7.20',
          androidGradlePlugin: '7.3.1'
        },
        '0.71': {
          gradle: '7.4.2',
          kotlin: '1.7.20',
          androidGradlePlugin: '7.3.0'
        },
        '0.70': {
          gradle: '7.3.1',
          kotlin: '1.7.10',
          androidGradlePlugin: '7.2.2'
        }
      };

      const majorMinor = rnVersion.split('.').slice(0, 2).join('.');
      return versionMap[majorMinor] || recommendedVersions;
    } catch (error) {
      console.log('Utilisation des versions par défaut'.warning);
      return {
        gradle: '8.3',
        kotlin: '1.9.20',
        androidGradlePlugin: '8.1.1'
      };
    }
  }

  static async updateGradleWrapper(androidDir, gradleVersion) {
    const wrapperPropsPath = path.join(
      androidDir,
      'gradle',
      'wrapper',
      'gradle-wrapper.properties',
    );

    if (!fs.existsSync(wrapperPropsPath)) {
      console.log('Fichier gradle-wrapper.properties non trouvé'.warning);
      return;
    }

    let content = fs.readFileSync(wrapperPropsPath, 'utf8');
    const newDistUrl = `https\\://services.gradle.org/distributions/gradle-${gradleVersion}-all.zip`;

    content = content.replace(
      /distributionUrl=.*gradle-.*-all\.zip/,
      `distributionUrl=${newDistUrl}`,
    );

    fs.writeFileSync(wrapperPropsPath, content);
    console.log(`gradle-wrapper.properties mis à jour avec Gradle ${gradleVersion}`.success);
  }

  static async updateProjectBuildGradle(
    androidDir,
    androidGradlePluginVersion,
    kotlinVersion,
  ) {
    const buildGradlePath = path.join(androidDir, 'build.gradle');

    if (!fs.existsSync(buildGradlePath)) {
      console.log('Fichier build.gradle (project) non trouvé'.warning);
      return;
    }

    let content = fs.readFileSync(buildGradlePath, 'utf8');

    content = content.replace(
      /classpath\("com\.android\.tools\.build:gradle:.*"\)/,
      `classpath("com.android.tools.build:gradle:${androidGradlePluginVersion}")`,
    );

    content = content.replace(
      /classpath\("org\.jetbrains\.kotlin:kotlin-gradle-plugin:.*"\)/,
      `classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:${kotlinVersion}")`,
    );

    fs.writeFileSync(buildGradlePath, content);
    console.log('build.gradle (project) mis à jour'.success);
  }

  static async updateGradleProperties(androidDir, versions) {
    const gradlePropsPath = path.join(androidDir, 'gradle.properties');
    
    if (!fs.existsSync(gradlePropsPath)) {
      console.log('Fichier gradle.properties non trouvé'.warning);
      return;
    }

    let content = fs.readFileSync(gradlePropsPath, 'utf8');
    
    if (!content.includes('kotlin.version')) {
      content += `\nkotlin.version=${versions.kotlin}\n`;
    } else {
      content = content.replace(
        /kotlin\.version=.*/,
        `kotlin.version=${versions.kotlin}`
      );
    }
    
    const recommendedConfigs = [
      'android.useAndroidX=true',
      'android.enableJetifier=true',
      'org.gradle.jvmargs=-Xmx2048m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8',
      'org.gradle.parallel=true',
      'org.gradle.daemon=true',
      'org.gradle.configureondemand=true'
    ];
    
    recommendedConfigs.forEach(config => {
      const [key] = config.split('=');
      const regex = new RegExp(`${key}=.*`);
      
      if (!content.includes(key)) {
        content += `\n${config}\n`;
      } else if (regex.test(content)) {
        content = content.replace(regex, config);
      }
    });

    content = content.replace(/org\.gradle\.jvmargs=.*-XX:MaxPermSize=\d+m/g, '');

    fs.writeFileSync(gradlePropsPath, content);
    console.log('gradle.properties mis à jour'.success);
  }
}

// Point d'entrée principal
async function main() {
  console.clear();
  UI.title('MISE À JOUR DE PROJET - DEEPSEEK');

  const packageManager = new PackageManager();
  const hasPackageJson = await packageManager.loadPackageJson();

  if (hasPackageJson) {
    EnvironmentManager.detectEnvironment(packageManager.packageData);
  }

  console.log('\nBonjour, choisissez une option :'.info.bold);
  console.log('1. Mettre à jour le package.json'.highlight);
  console.log('2. Installer des dépendances'.highlight);
  console.log('3. Comparer et mettre à jour les versions'.highlight);
  console.log('4. Structurer le projet'.highlight);
  console.log('5. Mettre à jour les fichiers Android'.highlight);

  const choice = await UI.ask('\nVotre choix (1-5) : ');

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
    case '4':
      await ProjectStructureManager.createStructure();
      break;
    case '5':
      await AndroidFilesUpdater.updateAndroidFiles();
      break;
    default:
      console.log('Option invalide'.error);
      process.exit(1);
  }

  await generateReport();
}

// Démarrer l'application
(async () => {
  try {
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
