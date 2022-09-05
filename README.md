# heroku-buildpack-lerna
A Heroku buildpack to deploy a single package of a monorepo application using [lerna](https://github.com/lerna/lerna) on Heroku.

This buildpack is based on another buildpack [heroku-buildpack-lerna](https://github.com/Zefir-Engineering/heroku-buildpack-lerna/blob/main/README.md), which
did not meet certain requirements, hence the need for another buildpack.

## Installation

- Add the default buildpack required for your desired package you want to deploy, e.g. `heroku/nodejs` for a node.js api
- Add the `heroku-buildpack-lerna` buildpack to your app: Go to your app, into the "Settings" tab, and click on "Add buildpack". Add `https://github.com/Julien-Ambos-Development/heroku-buildpack-lerna` as the buildpack url
- Make sure to set both buildpacks in the correct order. The `heroku-buildpack-lerna` needs to come last. This is required for `lerna` commands or others to be able to run

## Usage

To be able to deploy a single package, the following is required:

1. A Procfile containing your starting command, which needs to be placed inside the package you desire to deploy. See [how](#how) for further description.
2. A set of config variables need to be set on your Heroku app
3. Turn off production builds for `node`

### 1. Procfile

The Procfile is the entry point of an app in Heroku. Here, the commands necessary to run (not build) your app are specified.

E.g.:

```
web: node build/v1/server.js
```
which runs a node.js app using node with  `server.js` being the entry point of the application.

This file needs to be placed inside the folder of the package you desire to deploy. Please see [how](#how) for an example folder and file structure.

### <a name="config-variables"></a> 2. Config Variables

The following config variables must be defined in your app. To define config variables, go to the "Settings" tab of your app, click on "Reveal Config Vars" in the "Config Vars" section and add the following entries:

| Name | Description | Key | Value | Required |
|------|-------------|-----|-------|----------|
| Package name     | The fully qualified name of your package as stated in the package.json of your package (how this package would be referenced as a dependency)            | PACKAGE_NAME     | e.g. @my-company/example-1      |  Yes        |
| Package subfolder name     | The name of the subfolder inside your packages folder for your specific package            | PACKAGE_SUBFOLDER_NAME    | e.g. example1      | Yes         |
| Required local dependencies     | The required local package dependencies as a comma seperated list of {PackageName};{PackageSubFolderName}. Due to Heroku being unable to follow symlinks, this is required for local dependencies to be resolved. See [how](how) for further information.           | PACKAGE_REQUIRED_DEPENDENCIES     | e.g. @my-company/example-2;example2,@my-company/example-3;example3       | No        |
| Package path     | The folder name or path where your packages lay. Defaults to "packages".           |  PACKAGE_PATH      | e.g. packages         | No

### 3. Turn off production builds

For the `lerna` command to be available in this buildpack, the following Config Variable needs to be set

| Name | Description | Key | Value | Required |
|------|-------------|-----|-------|----------|
| Node environment     | Determines whether the build should be a production build or not    | NODE_ENV     | false      |  Yes        |

## <a name="how"></a>How does this buildpack work?

Given the following example folder structure:

```
- package.json
- lerna.json
- node_modules
- packages
    - example1
        - Procfile
        - package.json
    - example2
    - example3
```

and you desire to deploy the package "example1", this buildpack looks for a Procfile inside the packages/example1 directory. This Procfile will be used as the Procfile for the Heroku app.

If the "example1" package is a valid lerna package, the following commands will be executed in the scope of the package:

For the installation of required dependencies (in the current version of this buildpack, this step is unfortunately doubled with the previous running buildpack (feel free to submit a PR to optimize this):
```
npm ci && lerna bootstrap --ci
```

After the installation, the necessary packages are built with:
```
lerna run build --scope="${PACKAGE_NAME}" --include-dependencies --stream
```
with ${PACKAGE_NAME} being the name of the package as specified in the package.json of the package, e.g. "@mycompany/example-1". See [2. Config Variables](#config-variables) section for further reference.

Since lerna creates symlinks to link the local dependencies of a package, but Heroku does not follow symlinks, the symlinks are deleted and the built packages are moved to the `node_modules` folder of the "example1" package. 

Also during this step, the built "example1" package will be moved to the top level execution (build-output) folder of your app (this also includes the Procfile, therefore, the Procfile defined in the "example1" package folder will be used as your primary Procfile).

## Troubleshooting

- If lerna is not found, make sure to have lerna as a dev dependency in your package.json at the root of you repository.

## Support
For any inquiries, bugs or comments, please open an issue in this repository.