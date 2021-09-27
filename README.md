# SemVer + GitFlow + CommitLint
Repositório com passo-a-passo de como gerar um CHANGELOG automaticamente.

## Gerando um CHANGELOG automaticamente

### 1) Instale as dependências

```sh
npm install --save-dev husky @commitlint/cli @commitlint/config-conventional standard-version cz-conventional-changelog
```

### 2) Crie o arquivo `commitlint.config.js`

```js
module.exports = {
    extends: ['@commitlint/config-conventional']
};
```

### 3) Crie o arquivo `get-next-version.js`

```js
const packageJson = require('./package.json')
const conventionalRecommendedBump = require('conventional-recommended-bump')
const semver = require('semver')

const getNextVersion = currentVersion => {
  return new Promise((resolve, reject) => {
    conventionalRecommendedBump(
      {
        preset: 'angular',
      },
      (err, release) => {
        if (err) {
          reject(err)
          return
        }

        const nextVersion =
          semver.valid(release.releaseType) ||
          semver.inc(currentVersion, release.releaseType)

        resolve(nextVersion)
      }
    )
  })
}

getNextVersion(packageJson.version)
  .then(version => console.log(version))
  .catch(error => console.log(error))
```
### 4) Altere o seu `package.json`

```json
{
  ...,
  "husky": {
    "hooks": {
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS",
      "get-next-version": "node ./get-next-version.js"
    }
  },
  "scripts": {
    "release": "standard-version"
  },
   "config": {
    "commitizen": {
      "path": "./node_modules/cz-conventional-changelog"
    }
  },
  ...,
}
```

### 5) Crie o arquivo `.versionrc`

```json
{
  "types": [
    {"type": "feat", "section": "Funcionalidades"},
    {"type": "fix", "section": "Errors Corrigidos"},
    {"type": "chore", "hidden": true},
    {"type": "docs", "hidden": true},
    {"type": "style", "hidden": true},
    {"type": "refactor", "hidden": true},
    {"type": "perf", "section": "Melhorias de Performance"},
    {"type": "test", "hidden": true}
  ],
  "releaseCommitMessageFormat": "chore(release): {{currentTag}} [skip ci]"
}
```

### 5.1) Faça uma release inicial antes de automatizar (OPCIONAL)

Caso queira, pode criar uma release com a versão atual do projeto, antes de adicionar um processo de automatização.

```sh
npm run release
```

### 6) Crie o `.github/workflows/gerador-de-changelog.yml`

```yml
name: Gerador de CHANGELOG

# só executa no push de commit da branch master
on:
  push:
    branches:
      - master

jobs:
  build:
    # só deixa executar se o último commit não conter 'skip ci' na mensagem
    if: "!contains(github.event.head_commit.message, 'skip ci')"

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v1

    # configura o GITHUB_TOKEN
    # https://help.github.com/en/actions/automating-your-workflow-with-github-actions/authenticating-with-the-github_token
    - name: Configura o GitHub token
      uses: fregante/setup-git-token@v1
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        name: "Gerador de changelog"
        email: "changelog@users.noreply.github.com"

    # instala a versão 11.x do Node.js
    - name: Instala o Node.js 11.x
      uses: actions/setup-node@v1
      with:
        node-version: 11.x

    # instala as dependências, vai para a master
    # executa o standard-version, e envia o commit e tag
    - name: Gera o CHANGELOG
      run: |
        npm ci
        git checkout master
        npm run release
        git push origin master --follow-tags
```

### 7) Adicione a badge do workflow no `README.md`

```md
https://github.com/<OWNER>/<REPOSITORY>/workflows/<WORKFLOW_NAME>/badge.svg
```
