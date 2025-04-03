# Scripte-ReactNative
le nom du script est "Update_Projet_DeepSeek.js". pour le faire tourner, on fait "node Update_Projet_DeepSeek.js" à la racine du projet

#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const {execSync, exec} = require('child_process');
const readline = require('readline');
const {promisify} = require('util');
const os = require('os');
const semver = require('semver');
const fetch = require('node-fetch');
const opener = require('opener');
const moment = require('moment');

// Configuration
const CONFIG = {
  npmRegistry: 'https://registry.npmjs.org',
  npmsApi: 'https://api.npms.io/v2',
  reactNativeVersions:
    'https://raw.githubusercontent.com/react-native-community/rn-diff-purge/master/version-react-native.json',
  logDir: path.join(os.homedir(), 'npm_updater_logs'),
  maxParallelRequests: 5,
};

// Couleurs ANSI
const COLORS = {
  reset: '\x1b[0m',
  success: '\x1b[32m',
  warning: '\x1b[33m',
  error: '\x1b[31m',
  info: '\x1b[36m',
  highlight: '\x1b[1m\x1b[35m',
  muted: '\x1b[2m',
};

const style = (text, color) => `${color}${text}${COLORS.reset}`;

// Logs globaux
const GLOBAL_LOGS = {
  startTime: new Date(),
  operations: [],
  updatedPackages: [],
  skippedPackages: [],
  securityIssues: [],
  errors: [],
  createdFiles: [],
};

// Interface utilisateur améliorée
class UI {
  static async confirm(question, defaultValue = true) {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    });

    return new Promise(resolve => {
      const hint = defaultValue ? '(Y/n)' : '(y/N)';
      rl.question(
        `${style(question, COLORS.info)} ${style(hint, COLORS.muted)} `,
        answer => {
          rl.close();
          if (answer.trim() === '') return resolve(defaultValue);
          resolve(/^y(es)?$/i.test(answer.trim()));
        },
      );
    });
  }

  static async ask(question) {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    });

    return new Promise(resolve => {
      rl.question(style(question + ' ', COLORS.info), answer => {
        rl.close();
        resolve(answer.trim());
      });
    });
  }

  static progress(current, total, message = '') {
    const percent = Math.round((current / total) * 100);
    const progressBar = `[${'='.repeat(Math.floor(percent / 5))}${' '.repeat(
      20 - Math.floor(percent / 5),
    )}]`;
    process.stdout.clearLine();
    process.stdout.cursorTo(0);
    process.stdout.write(
      `${style(message, COLORS.info)} ${progressBar} ${percent}%`,
    );
  }

  static completeProgress(message) {
    process.stdout.clearLine();
    process.stdout.cursorTo(0);
    console.log(style(message, COLORS.success));
  }

  static separator() {
    console.log(style('━'.repeat(50), COLORS.muted));
  }

  static title(message) {
    console.log('\n');
    UI.separator();
    console.log(style(` ${message} `, COLORS.highlight));
    UI.separator();
  }

  static showCheck(message, condition) {
    console.log(
      ` ${
        condition ? style('✓', COLORS.success) : style('✗', COLORS.error)
      } ${message}`,
    );
  }
}

// Gestionnaire de fichiers
class FileManager {
  static ensureLogsDirectory() {
    if (!fs.existsSync(CONFIG.logDir)) {
      fs.mkdirSync(CONFIG.logDir, {recursive: true});
    }
  }

  static async saveLogFile(content) {
    this.ensureLogsDirectory();
    const timestamp = moment().format('YYYY-MM-DD_HH-mm-ss');
    const logFileName = `npm_update_${timestamp}.log`;
    const logPath = path.join(CONFIG.logDir, logFileName);

    await promisify(fs.writeFile)(logPath, content);
    return logPath;
  }

  static findPackageJson() {
    const files = fs.readdirSync(process.cwd());
    return files.find(file => file.toLowerCase() === 'package.json');
  }

  static async createProjectStructure(structure) {
    const lines = structure.split('\n');
    for (const line of lines) {
      const trimmed = line.trim();
      if (trimmed) {
        const isFile = trimmed.includes('.');
        const fullPath = path.join(process.cwd(), trimmed);

        try {
          if (isFile) {
            const dir = path.dirname(fullPath);
            if (!fs.existsSync(dir)) {
              fs.mkdirSync(dir, {recursive: true});
            }
            fs.writeFileSync(fullPath, '');
            GLOBAL_LOGS.createdFiles.push(`Fichier créé: ${fullPath}`);
          } else {
            fs.mkdirSync(fullPath, {recursive: true});
            GLOBAL_LOGS.createdFiles.push(`Dossier créé: ${fullPath}`);
          }
        } catch (error) {
          GLOBAL_LOGS.errors.push(
            `Erreur création ${isFile ? 'fichier' : 'dossier'} ${trimmed}: ${
              error.message
            }`,
          );
        }
      }
    }
  }
}

// Gestionnaire de dépendances
class DependencyManager {
  constructor() {
    this.packageJsonPath = null;
    this.packageData = null;
    this.updateStrategy = 1;
    this.apiClient = new ApiClient();
  }

  async initialize() {
    await this.handlePackageJson();
    await this.handleProjectStructure();
  }

  async handlePackageJson() {
    const existingPackageJson = FileManager.findPackageJson();

    if (existingPackageJson) {
      UI.title('DÉTECTION DE PACKAGE.JSON');
      console.log(
        style(
          `Un fichier package.json existe déjà dans ${process.cwd()}`,
          COLORS.info,
        ),
      );

      const useExisting = await UI.confirm(
        "Voulez-vous l'utiliser pour la mise à jour?",
      );
      if (!useExisting) {
        const useCustom = await UI.confirm(
          'Voulez-vous fournir un package.json personnalisé?',
        );
        if (useCustom) {
          await this.getCustomPackageJson();
        } else {
          console.log(style('Opération annulée.', COLORS.error));
          process.exit(0);
        }
      } else {
        this.packageJsonPath = path.join(process.cwd(), existingPackageJson);
        this.loadPackageJson();
      }
    } else {
      console.log(
        style(
          'Aucun package.json trouvé dans le répertoire courant.',
          COLORS.warning,
        ),
      );
      const useCustom = await UI.confirm(
        'Voulez-vous fournir un package.json personnalisé?',
      );
      if (useCustom) {
        await this.getCustomPackageJson();
      } else {
        console.log(style('Opération annulée.', COLORS.error));
        process.exit(0);
      }
    }
  }

  async getCustomPackageJson() {
    console.log(
      style(
        '\nCollez le contenu de votre package.json (appuyez sur Entrée puis Ctrl+D pour terminer):',
        COLORS.info,
      ),
    );

    let input = '';
    process.stdin.setEncoding('utf8');
    process.stdin.on('data', chunk => {
      input += chunk;
    });

    await new Promise(resolve => {
      process.stdin.on('end', () => {
        try {
          this.packageData = JSON.parse(input);
          console.log(
            style(
              'Package.json personnalisé chargé avec succès!',
              COLORS.success,
            ),
          );
          resolve();
        } catch (e) {
          console.log(
            style(
              'Erreur de parsing du JSON. Opération annulée.',
              COLORS.error,
            ),
          );
          process.exit(1);
        }
      });

      // Pour permettre à l'utilisateur de terminer la saisie
      process.stdin.on('close', resolve);
    });
  }

  async handleProjectStructure() {
    const hasStructure = await UI.confirm(
      '\nAvez-vous une structure spécifique pour votre application?',
    );
    if (hasStructure) {
      console.log(
        style(
          '\nCollez la structure de votre projet (ex: src/\n  components/\n    Button.js\n  assets/\n    images/)',
          COLORS.info,
        ),
      );
      console.log(
        style('Appuyez sur Entrée puis Ctrl+D pour terminer:', COLORS.muted),
      );

      let structure = '';
      process.stdin.setEncoding('utf8');
      process.stdin.on('data', chunk => {
        structure += chunk;
      });

      await new Promise(resolve => {
        process.stdin.on('end', () => {
          FileManager.createProjectStructure(structure);
          if (GLOBAL_LOGS.createdFiles.length > 0) {
            console.log(
              style('\nStructure créée avec succès:', COLORS.success),
            );
            GLOBAL_LOGS.createdFiles.forEach(item => console.log(` - ${item}`));
          }
          resolve();
        });
        process.stdin.on('close', resolve);
      });
    }
  }

  loadPackageJson() {
    try {
      const content = fs.readFileSync(this.packageJsonPath, 'utf8');
      this.packageData = JSON.parse(content);
      GLOBAL_LOGS.originalPackage = JSON.parse(
        JSON.stringify(this.packageData),
      );
      console.log(style('Package.json chargé avec succès!', COLORS.success));
    } catch (error) {
      console.log(style('Erreur de lecture du package.json', COLORS.error));
      process.exit(1);
    }
  }

  async checkDependencies() {
    UI.title('VÉRIFICATION DES DÉPENDANCES');

    this.updateStrategy = await this.askUpdateStrategy();
    if (this.updateStrategy === 3) {
      console.log(style("Opération annulée par l'utilisateur.", COLORS.info));
      process.exit(0);
    }

    await this.checkNodeVersion();
    await this.checkReactNativeCompatibility();
    await this.checkGradleCompatibility();
    await this.checkDependencyCategory('dependencies');
    await this.checkDependencyCategory('devDependencies');
    await this.checkPeerDependencies();
  }

  async askUpdateStrategy() {
    console.log('\n');
    console.log(style('Stratégie de mise à jour:', COLORS.info));
    console.log(' 1) Mise à jour globale automatique');
    console.log(' 2) Confirmation pour chaque package');
    console.log(' 3) Annuler');

    const choice = await UI.ask(style('\nVotre choix (1-3):', COLORS.info));
    return parseInt(choice.trim(), 10) || 3;
  }

  async checkNodeVersion() {
    UI.title('VÉRIFICATION DE NODE.JS');

    try {
      const currentNode = process.version;
      const recommendedNode = await this.getRecommendedNodeVersion();

      console.log(`Version actuelle: ${style(currentNode, COLORS.info)}`);
      console.log(
        `Version recommandée: ${style(recommendedNode, COLORS.info)}`,
      );

      if (!semver.satisfies(currentNode, recommendedNode)) {
        console.log(style('⚠️ Version Node.js incompatible', COLORS.warning));
        const shouldUpdate = await UI.confirm(
          'Voulez-vous mettre à jour Node.js?',
        );
        if (shouldUpdate) {
          this.updateNodeVersion(recommendedNode);
        }
      } else {
        console.log(style('✅ Version Node.js compatible', COLORS.success));
      }
    } catch (error) {
      console.log(
        style('⚠️ Impossible de vérifier la version Node.js', COLORS.error),
      );
      GLOBAL_LOGS.errors.push(`Node.js version check failed: ${error.message}`);
    }
  }

  async getRecommendedNodeVersion() {
    try {
      const response = await fetch(CONFIG.reactNativeVersions);
      const data = await response.json();
      return data.recommendedNodeVersion || '>=14';
    } catch {
      return '>=14';
    }
  }

  updateNodeVersion(version) {
    try {
      console.log(style('\nOptions de mise à jour:', COLORS.info));
      console.log('1) Utiliser nvm (recommandé)');
      console.log('2) Télécharger manuellement');

      const choice = UI.ask('Votre choix (1-2):');
      if (choice === '1') {
        execSync(`nvm install ${version}`, {stdio: 'inherit'});
      } else {
        opener('https://nodejs.org/');
        console.log(
          style(
            '\nTéléchargez la version recommandée sur le site officiel',
            COLORS.info,
          ),
        );
      }
    } catch (error) {
      console.log(style('Échec de la mise à jour de Node.js', COLORS.error));
      GLOBAL_LOGS.errors.push(`Failed to update Node.js: ${error.message}`);
    }
  }

  async checkReactNativeCompatibility() {
    if (
      !this.packageData.dependencies ||
      !this.packageData.dependencies['react-native']
    ) {
      return;
    }

    UI.title('VÉRIFICATION REACT NATIVE');

    try {
      const currentRN = this.packageData.dependencies['react-native'];
      const currentReact = this.packageData.dependencies.react;

      const [latestRN, latestReact] = await Promise.all([
        this.apiClient.getLatestVersion('react-native'),
        this.apiClient.getLatestVersion('react'),
      ]);

      console.log(
        `React Native: ${style(currentRN, COLORS.info)} → ${style(
          latestRN,
          COLORS.info,
        )}`,
      );
      console.log(
        `React: ${style(currentReact, COLORS.info)} → ${style(
          latestReact,
          COLORS.info,
        )}`,
      );

      const shouldUpdate = await UI.confirm(
        'Voulez-vous mettre à jour React Native et React?',
      );
      if (shouldUpdate) {
        execSync(
          `npm install react-native@${latestRN} react@${latestReact} --legacy-peer-deps`,
          {
            stdio: 'inherit',
          },
        );
        console.log(
          style('✅ React Native et React mis à jour', COLORS.success),
        );
      }
    } catch (error) {
      console.log(
        style('⚠️ Erreur de vérification React Native', COLORS.error),
      );
      GLOBAL_LOGS.errors.push(`React Native check failed: ${error.message}`);
    }
  }

  async checkGradleCompatibility() {
    if (!fs.existsSync(path.join(process.cwd(), 'android'))) {
      return;
    }

    UI.title('VÉRIFICATION GRADLE');

    try {
      const wrapperPropsPath = path.join(
        process.cwd(),
        'android',
        'gradle',
        'wrapper',
        'gradle-wrapper.properties',
      );
      const buildGradlePath = path.join(
        process.cwd(),
        'android',
        'build.gradle',
      );
      const appBuildGradlePath = path.join(
        process.cwd(),
        'android',
        'app',
        'build.gradle',
      );

      if (fs.existsSync(wrapperPropsPath)) {
        const wrapperProps = fs.readFileSync(wrapperPropsPath, 'utf8');
        const gradleVersionMatch = wrapperProps.match(
          /gradle-(\d+\.\d+(\.\d+)?)-/,
        );
        if (gradleVersionMatch) {
          const currentGradle = gradleVersionMatch[1];
          const recommendedGradle = await this.getRecommendedGradleVersion();

          console.log(
            `Gradle: ${style(currentGradle, COLORS.info)} (recommandé: ${style(
              recommendedGradle,
              COLORS.info,
            )})`,
          );

          if (semver.lt(currentGradle, recommendedGradle)) {
            const shouldUpdate = await UI.confirm(
              'Voulez-vous mettre à jour Gradle?',
            );
            if (shouldUpdate) {
              this.updateGradleVersion(
                wrapperPropsPath,
                buildGradlePath,
                appBuildGradlePath,
                recommendedGradle,
              );
            }
          } else {
            console.log(style('✅ Version Gradle compatible', COLORS.success));
          }
        }
      }
    } catch (error) {
      console.log(style('⚠️ Erreur de vérification Gradle', COLORS.error));
      GLOBAL_LOGS.errors.push(`Gradle check failed: ${error.message}`);
    }
  }

  async getRecommendedGradleVersion() {
    try {
      const cliVersion =
        this.packageData.devDependencies['@react-native-community/cli'] ||
        '^5.0.0';
      const majorVersion = semver.major(semver.coerce(cliVersion));

      if (majorVersion >= 7) return '7.5.1';
      if (majorVersion >= 5) return '6.9';
      return '6.3';
    } catch {
      return '7.5.1';
    }
  }

  updateGradleVersion(wrapperPath, buildPath, appBuildPath, version) {
    try {
      // Mise à jour gradle-wrapper.properties
      let wrapperContent = fs.readFileSync(wrapperPath, 'utf8');
      wrapperContent = wrapperContent.replace(
        /distributionUrl=.*gradle-.*?-all\.zip/,
        `distributionUrl=https\\://services.gradle.org/distributions/gradle-${version}-all.zip`,
      );
      fs.writeFileSync(wrapperPath, wrapperContent);

      // Mise à jour build.gradle
      if (fs.existsSync(buildPath)) {
        let buildContent = fs.readFileSync(buildPath, 'utf8');
        buildContent = buildContent.replace(
          /classpath\("com\.android\.tools\.build:gradle:.*?"\)/,
          `classpath("com.android.tools.build:gradle:7.2.0")`,
        );
        fs.writeFileSync(buildPath, buildContent);
      }

      // Mise à jour app/build.gradle
      if (fs.existsSync(appBuildPath)) {
        let appBuildContent = fs.readFileSync(appBuildPath, 'utf8');
        appBuildContent = appBuildContent
          .replace(/compileSdkVersion = \d+/, 'compileSdkVersion = 31')
          .replace(/targetSdkVersion = \d+/, 'targetSdkVersion = 31');
        fs.writeFileSync(appBuildPath, appBuildContent);
      }

      console.log(style('✅ Gradle mis à jour avec succès', COLORS.success));
    } catch (error) {
      console.log(
        style('⚠️ Erreur lors de la mise à jour de Gradle', COLORS.error),
      );
      GLOBAL_LOGS.errors.push(`Gradle update failed: ${error.message}`);
    }
  }

  async checkDependencyCategory(category) {
    if (!this.packageData[category]) {
      return;
    }

    UI.title(`VÉRIFICATION DES ${category.toUpperCase()}`);

    const packages = Object.entries(this.packageData[category]);
    const total = packages.length;
    let updatedCount = 0;

    console.log(style(`Analyse de ${total} packages...\n`, COLORS.info));

    // Utilisation de Promise.allSettled pour paralléliser les requêtes
    const results = await Promise.allSettled(
      packages.map(async ([pkg, version]) => {
        try {
          const latestVersion = await this.apiClient.getLatestVersion(pkg);
          return {pkg, currentVersion: version, latestVersion};
        } catch (error) {
          GLOBAL_LOGS.errors.push(`Failed to get version for ${pkg}`);
          return {
            pkg,
            currentVersion: version,
            latestVersion: null,
            error: true,
          };
        }
      }),
    );

    for (const [index, result] of results.entries()) {
      UI.progress(
        index + 1,
        total,
        `Traitement de ${result.value?.pkg || packages[index][0]}`,
      );

      if (result.status === 'rejected' || result.value.error) {
        UI.showCheck(
          `${style(
            result.value?.pkg || packages[index][0],
            COLORS.error,
          )} - erreur de vérification`,
          false,
        );
        continue;
      }

      const {pkg, currentVersion, latestVersion} = result.value;

      if (!latestVersion) {
        UI.showCheck(
          `${style(pkg, COLORS.error)} - version non trouvée`,
          false,
        );
        continue;
      }

      const isUpToDate = semver.satisfies(
        semver.coerce(latestVersion),
        currentVersion.replace(/^[\^~]/, ''),
      );

      if (isUpToDate) {
        GLOBAL_LOGS.skippedPackages.push({pkg, currentVersion, latestVersion});
        UI.showCheck(
          `${style(pkg, COLORS.success)} est à jour (${style(
            currentVersion,
            COLORS.info,
          )})`,
          true,
        );
        continue;
      }

      console.log(
        `\n${style('!', COLORS.warning)} ${style(
          pkg,
          COLORS.highlight,
        )} peut être mis à jour:`,
      );
      console.log(`  Version actuelle: ${style(currentVersion, COLORS.info)}`);
      console.log(
        `  Version disponible: ${style(latestVersion, COLORS.success)}`,
      );

      let shouldUpdate = this.updateStrategy === 1;
      if (this.updateStrategy === 2) {
        shouldUpdate = await UI.confirm(`Mettre à jour ${pkg}?`, true);
      }

      if (shouldUpdate) {
        try {
          await this.installPackage(
            pkg,
            latestVersion,
            category === 'devDependencies',
          );
          updatedCount++;
          GLOBAL_LOGS.updatedPackages.push({
            pkg,
            from: currentVersion,
            to: latestVersion,
          });
          UI.showCheck(
            `${style(pkg, COLORS.success)} mis à jour avec succès`,
            true,
          );
        } catch (error) {
          await this.handlePackageError(pkg, currentVersion);
        }
      } else {
        GLOBAL_LOGS.skippedPackages.push({pkg, currentVersion, latestVersion});
      }
    }

    UI.completeProgress(
      `\n${category}: ${updatedCount} packages mis à jour sur ${total}`,
    );
  }

  async installPackage(pkg, version, isDev = false) {
    return new Promise((resolve, reject) => {
      const command = `npm install ${pkg}@${version} --save-exact --legacy-peer-deps${
        isDev ? ' --save-dev' : ''
      }`;
      const child = exec(command, {stdio: 'inherit'});

      child.on('exit', code => {
        if (code === 0) {
          resolve();
        } else {
          reject(new Error(`Installation failed with code ${code}`));
        }
      });
    });
  }

  async checkPeerDependencies() {
    if (!this.packageData.peerDependencies) {
      return;
    }

    UI.title('VÉRIFICATION DES PEER DEPENDENCIES');

    const peers = Object.entries(this.packageData.peerDependencies);
    console.log(
      style(`Analyse de ${peers.length} peerDependencies...\n`, COLORS.info),
    );

    for (const [pkg, range] of peers) {
      try {
        const installedVersion = this.getInstalledVersion(pkg);
        if (!installedVersion) {
          console.log(
            style(
              `⚠️ ${pkg} n'est pas installé mais requis comme peerDependency`,
              COLORS.warning,
            ),
          );
          const shouldInstall = await UI.confirm(
            `Voulez-vous installer ${pkg}?`,
          );
          if (shouldInstall) {
            await this.installPackage(pkg, 'latest');
          }
          continue;
        }

        if (!semver.satisfies(semver.coerce(installedVersion), range)) {
          console.log(
            style(
              `⚠️ Version incompatible de ${pkg}: ${installedVersion} (requis: ${range})`,
              COLORS.warning,
            ),
          );
          const shouldUpdate = await UI.confirm(
            `Mettre à jour ${pkg} pour satisfaire la peerDependency?`,
          );
          if (shouldUpdate) {
            await this.installPackage(pkg, range);
          }
        }
      } catch (error) {
        GLOBAL_LOGS.errors.push(
          `Failed to check peerDependency ${pkg}: ${error.message}`,
        );
      }
    }
  }

  getInstalledVersion(pkg) {
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

  async handlePackageError(pkg, currentVersion) {
    GLOBAL_LOGS.errors.push(`Échec de la mise à jour de ${pkg}`);
    UI.showCheck(
      `Échec de la mise à jour de ${style(pkg, COLORS.error)}`,
      false,
    );

    const alternative = await this.apiClient.getAlternativePackage(pkg);
    if (alternative) {
      console.log(style(`\nAlternative proposée pour ${pkg}:`, COLORS.info));
      console.log(`  Nom: ${style(alternative.name, COLORS.highlight)}`);
      console.log(`  Description: ${alternative.description}`);
      console.log(`  Score: ${alternative.score.toFixed(2)}`);

      const useAlternative = await UI.confirm(
        `Voulez-vous installer ${alternative.name} à la place?`,
      );
      if (useAlternative) {
        try {
          await this.installPackage(alternative.name, 'latest');
          GLOBAL_LOGS.updatedPackages.push({
            pkg: alternative.name,
            from: `(remplacement de ${pkg}@${currentVersion})`,
            to: 'latest',
          });
          console.log(
            style(
              `✅ ${alternative.name} installé avec succès`,
              COLORS.success,
            ),
          );
        } catch (error) {
          console.log(
            style(
              `❌ Échec de l'installation de ${alternative.name}`,
              COLORS.error,
            ),
          );
        }
      }
    }
  }

  async generateReport() {
    UI.title('RAPPORT FINAL');

    // Packages mis à jour
    if (GLOBAL_LOGS.updatedPackages.length > 0) {
      console.log(style('\nPACKAGES MIS À JOUR:', COLORS.success));
      GLOBAL_LOGS.updatedPackages.forEach(({pkg, from, to}) => {
        console.log(
          ` ${style('✓', COLORS.success)} ${style(
            pkg,
            COLORS.highlight,
          )}: ${from} → ${to}`,
        );
      });
    }

    // Packages déjà à jour
    if (GLOBAL_LOGS.skippedPackages.length > 0) {
      console.log(style('\nPACKAGES DÉJÀ À JOUR:', COLORS.info));
      GLOBAL_LOGS.skippedPackages
        .slice(0, 10)
        .forEach(({pkg, currentVersion}) => {
          console.log(
            ` ${style('✓', COLORS.success)} ${style(
              pkg,
              COLORS.highlight,
            )} @ ${currentVersion}`,
          );
        });
      if (GLOBAL_LOGS.skippedPackages.length > 10) {
        console.log(
          style(
            ` (+ ${GLOBAL_LOGS.skippedPackages.length - 10} autres packages)`,
            COLORS.muted,
          ),
        );
      }
    }

    // Fichiers créés
    if (GLOBAL_LOGS.createdFiles.length > 0) {
      console.log(style('\nSTRUCTURE CRÉÉE:', COLORS.info));
      GLOBAL_LOGS.createdFiles.forEach(file => {
        console.log(` ${style('✓', COLORS.success)} ${file}`);
      });
    }

    // Erreurs
    if (GLOBAL_LOGS.errors.length > 0) {
      console.log(style('\nERREURS RENCONTRÉES:', COLORS.error));
      const uniqueErrors = [...new Set(GLOBAL_LOGS.errors)];
      uniqueErrors.slice(0, 20).forEach((error, i) => {
        console.log(` ${i + 1}) ${error}`);
      });
      if (uniqueErrors.length > 20) {
        console.log(
          style(
            ` (+ ${uniqueErrors.length - 20} autres erreurs)`,
            COLORS.muted,
          ),
        );
      }
    }

    // Statistiques
    const duration = moment.duration(moment().diff(GLOBAL_LOGS.startTime));
    console.log(style('\nSTATISTIQUES:', COLORS.highlight));
    console.log(` Durée totale: ${duration.minutes()}m ${duration.seconds()}s`);
    console.log(` Packages mis à jour: ${GLOBAL_LOGS.updatedPackages.length}`);
    console.log(` Packages déjà à jour: ${GLOBAL_LOGS.skippedPackages.length}`);
    console.log(` Fichiers/dossiers créés: ${GLOBAL_LOGS.createdFiles.length}`);
    console.log(` Erreurs: ${GLOBAL_LOGS.errors.length}`);

    await this.saveDetailedLog();
  }

  async saveDetailedLog() {
    const logContent = `
=== RAPPORT DE MISE À JOUR NPM ===
Date: ${GLOBAL_LOGS.startTime.toISOString()}
Durée: ${moment.duration(moment().diff(GLOBAL_LOGS.startTime)).humanize()}
Répertoire: ${process.cwd()}

=== PACKAGES MIS À JOUR ===
${
  GLOBAL_LOGS.updatedPackages
    .map(p => `${p.pkg}: ${p.from} → ${p.to}`)
    .join('\n') || 'Aucun'
}

=== PACKAGES DÉJÀ À JOUR ===
${
  GLOBAL_LOGS.skippedPackages
    .map(p => `${p.pkg} @ ${p.currentVersion}`)
    .join('\n') || 'Aucun'
}

=== STRUCTURE CRÉÉE ===
${GLOBAL_LOGS.createdFiles.join('\n') || 'Aucune'}

=== ERREURS ===
${[...new Set(GLOBAL_LOGS.errors)].join('\n') || 'Aucune'}

=== ANCIEN PACKAGE.JSON ===
${JSON.stringify(GLOBAL_LOGS.originalPackage, null, 2) || 'Non disponible'}

=== NOUVEAU PACKAGE.JSON ===
${JSON.stringify(this.packageData, null, 2) || 'Non disponible'}
        `;

    const logPath = await FileManager.saveLogFile(logContent);
    console.log(
      style(
        `\nUn fichier log détaillé a été sauvegardé ici:\n${style(
          logPath,
          COLORS.highlight,
        )}`,
        COLORS.info,
      ),
    );

    const openLog = await UI.confirm(
      'Voulez-vous ouvrir le fichier log maintenant?',
      false,
    );
    if (openLog) {
      try {
        opener(logPath);
      } catch {
        console.log(
          style(
            "Impossible d'ouvrir le fichier automatiquement.",
            COLORS.muted,
          ),
        );
      }
    }
  }
}

// Client API pour npm
class ApiClient {
  constructor() {
    this.cache = new Map();
  }

  async getLatestVersion(packageName) {
    if (this.cache.has(packageName)) {
      return this.cache.get(packageName);
    }

    try {
      const response = await fetch(
        `${CONFIG.npmRegistry}/${packageName}/latest`,
      );
      if (!response.ok) throw new Error('Package not found');
      const data = await response.json();
      this.cache.set(packageName, data.version);
      return data.version;
    } catch (error) {
      throw new Error(`Failed to get version for ${packageName}`);
    }
  }

  async getAlternativePackage(packageName) {
    try {
      const response = await fetch(`${CONFIG.npmsApi}/search?q=${packageName}`);
      const data = await response.json();
      if (data.results.length > 0) {
        return {
          name: data.results[0].package.name,
          description: data.results[0].package.description,
          score: data.results[0].score.final,
        };
      }
      return null;
    } catch {
      return null;
    }
  }
}

// Fonction principale
async function main() {
  try {
    console.clear();
    UI.title('NPM PACKAGE UPDATER - OUTIL DE MISE À JOUR INTELLIGENT');

    const manager = new DependencyManager();
    await manager.initialize();
    await manager.checkDependencies();
    await manager.generateReport();

    console.log('\n');
    UI.separator();
    console.log(style(' Opération terminée avec succès! ', COLORS.success));
    UI.separator();
  } catch (error) {
    console.error(style('\nERREUR CRITIQUE:', COLORS.error), error);
    process.exit(1);
  }
}

// Vérification des dépendances requises
async function checkDependencies() {
  try {
    // Vérifier si les dépendances sont installées
    require('semver');
    require('node-fetch');
    require('moment');
  } catch (err) {
    console.log(style('Installation des dépendances requises...', COLORS.info));
    try {
      execSync(
        'npm install semver node-fetch@2 moment opener --no-save --legacy-peer-deps',
        {stdio: 'inherit'},
      );
    } catch (error) {
      console.log(
        style(
          "Échec de l'installation des dépendances. Exécutez manuellement:",
          COLORS.error,
        ),
      );
      console.log(
        style(
          'npm install semver node-fetch@2 moment opener --no-save --legacy-peer-deps',
          COLORS.highlight,
        ),
      );
      process.exit(1);
    }
  }
}

// Démarrer l'application
checkDependencies().then(main);
