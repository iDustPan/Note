
## A Project with webpack and React

### webpack step

`npm init -y`

after it, open the `package.json` file

update follow:

```
"author": "Pan",
"license": "MIT",
```

then install webpack use follow command:

`npm install --save-dev webpack webpack-dev-server`

for more details, can read this document: [webpack document](https://webpack.js.org/concepts/)

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

configure the `package.json` file:

```
...
"scripts": {
    "build": "webpack",
    "start": "webpack-dev-server --open"
  },
...

```

### React step

input follow command in terminal at project path:

`npm install --save react react-dom`

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


Next, you should touch a `index.html` at the root directory, add this code in it:

```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>React Setup</title>
    </head>
    <body>
        <div id="app"></div>
        <script src="./bundle.js"></script>
    </body>
</html>
```

then touch a file `index.js` in `./src/`, write the hello world code:

```

import React from "react"
import ReactDom from 'react-dom'

class App extends React.Component {
    render() {
        return <h1>Hello world!</h1>;
    }
}

ReactDom.render(
    <App />,
    document.getElementById('app')
);

```

open the console,input `npm start`, then, happy hacking...

### what`s the difference between '--save-dev' and '--save'?

- `--save-dev` is used to save the package for development purpose. Example: unit tests, minification..

- `--save` is used to save the package required for the application to run.
