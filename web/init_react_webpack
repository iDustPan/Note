
## A Project with Webpack and React

### Webpack step

`npm init -y`

after it, open the `package.json` file

update follow:

```
"author": "Pan",
"license": "MIT",
```

then install webpack use follow command:

`npm install --save-dev webpack`

touch file `webpack.config.js` and append follow config.

```
module.exports = {
    entry: "./src/index.js",
    output: {
        path: __dirname,
        publicPath: '/',
        filename: 'bundle.js'
    },
    devServer: {
        contentBase: './'
    },
};
```

### React step

input follow command in terminal at project path:

`npm install --save-dev react react-dom`

then add babel dependencies:

`npm install --save-dev babel-preset-react babel-preset-stage-2 babel-preset-env babel-loader babel-core`

then append this code at `webpack.config.js`:

```
module.exports = {
    ...

    module: {
      rules: [
        { test: /\.js$/, exclude: /node_modules/, loader: "babel-loader" }
      ]
    }
};

```

then you can run this command to add `.babelrc` file

`echo '{ "presets": ["react", "env" , "stage-2"] }' > .babelrc`

thus, a simple Webpack react project has been initialized.

### what`s the difference between '--save-dev' and '--save'?

- `--save-dev` is used to save the package for development purpose. Example: unit tests, minification..

- `--save` is used to save the package required for the application to run.
