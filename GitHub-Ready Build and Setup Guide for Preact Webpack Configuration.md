# Build and Setup Guide for Preact Webpack Configuration

## Table of Contents

1. [Introduction](#introduction)
2. [Summary](#summary)
3. [Full Configuration](#full-configuration)
   - [project.config.js](#projectconfigjs)
   - [webpack.common.js](#webpackcommonjs)
   - [webpack.dev.js](#webpackdevjs)
   - [webpack.prod.js](#webpackprodjs)
   - [tsconfig.json](#tsconfigjson)
   - [.eslintrc.js](#eslintrcjs)
   - [package.json](#packagejson)
4. [Setup Instructions](#setup-instructions)
5. [API](#api)
6. [FAQ](#faq)
7. [Conclusion](#conclusion)

## Introduction

This guide provides a comprehensive webpack configuration for Preact projects, optimized for performance, developer experience, and modern web development practices. Our setup includes support for TypeScript, CSS processing, image optimization, internationalization, and more.

## Summary

Our webpack configuration is designed to:
- Optimize for Preact development
- Support TypeScript
- Process and optimize CSS
- Handle and optimize images
- Support internationalization with preact-i18n
- Implement CSS-in-JS with goober
- Provide a robust development environment with hot reloading
- Optimize production builds for performance

## Full Configuration

### project.config.js

```javascript
const path = require('path');

module.exports = {
  projectName: 'My Preact App',
  projectDescription: 'A fast and lightweight Preact application',
  srcDir: path.resolve(__dirname, 'src'),
  publicDir: path.resolve(__dirname, 'public'),
  outputDir: path.resolve(__dirname, 'dist'),
  entry: path.resolve(__dirname, 'src/index.js'),
  htmlTemplate: path.resolve(__dirname, 'public/index.html'),
  favicon: path.resolve(__dirname, 'src/assets/favicon.png'),
  devServerPort: 3000,
  devServerProxy: {
    '/api': 'http://localhost:8080',
  },
  aliases: {
    '@': path.resolve(__dirname, 'src'),
    '@components': path.resolve(__dirname, 'src/components'),
    '@styles': path.resolve(__dirname, 'src/styles'),
  },
  images: {
    sizes: [300, 600, 1200, 2000],
    placeholderSize: 40,
    outputPath: 'images/',
    quality: 80,
  },
};
```

### webpack.common.js

```javascript
const webpack = require('webpack');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const CopyWebpackPlugin = require('copy-webpack-plugin');
const FaviconsWebpackPlugin = require('favicons-webpack-plugin');
const Dotenv = require('dotenv-webpack');
const ESLintPlugin = require('eslint-webpack-plugin');
const config = require('../project.config');

module.exports = {
  entry: {
    main: config.entry,
  },
  output: {
    path: config.outputDir,
    filename: '[name].[contenthash].js',
    publicPath: '/',
  },
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx', '.json'],
    alias: {
      ...config.aliases,
      'react': 'preact/compat',
      'react-dom': 'preact/compat',
    },
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', { targets: { browsers: ['last 2 versions', 'not dead', 'not < 2%'] } }],
              ['@babel/preset-react', { pragma: 'h', pragmaFrag: 'Fragment' }]
            ],
            plugins: [
              '@babel/plugin-proposal-class-properties',
              '@babel/plugin-syntax-dynamic-import',
              ['@babel/plugin-transform-react-jsx', { pragma: 'h', pragmaFrag: 'Fragment' }]
            ],
          },
        },
      },
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader'],
      },
      {
        test: /\.(png|jpe?g|gif|svg|webp)$/i,
        use: [
          {
            loader: 'responsive-loader',
            options: {
              adapter: require('responsive-loader/sharp'),
              sizes: config.images.sizes,
              placeholder: true,
              placeholderSize: config.images.placeholderSize,
              name: `${config.images.outputPath}[name]-[width]-[hash:8].[ext]`,
              format: 'webp',
              quality: config.images.quality,
            },
          },
        ],
      },
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource',
      },
    ],
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({
      template: config.htmlTemplate,
      title: config.projectName,
      meta: {
        description: config.projectDescription,
      },
    }),
    new CopyWebpackPlugin({
      patterns: [
        { 
          from: config.publicDir, 
          to: config.outputDir,
          globOptions: { ignore: ['**/index.html'] }
        },
      ],
    }),
    new FaviconsWebpackPlugin({
      logo: config.favicon,
      mode: 'webapp',
      devMode: 'webapp',
      favicons: {
        appName: config.projectName,
        appDescription: config.projectDescription,
        background: '#ddd',
        theme_color: '#333',
      },
    }),
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
    }),
    new Dotenv({
      path: `./.env.${process.env.NODE_ENV}`,
    }),
    new ESLintPlugin({
      extensions: ['js', 'jsx', 'ts', 'tsx'],
      exclude: ['node_modules'],
    }),
  ],
};
```

### webpack.dev.js

```javascript
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
  mode: 'development',
  devtool: 'eval-source-map',
  devServer: {
    historyApiFallback: true,
    hot: true,
    open: true,
    port: 3000,
    host: '0.0.0.0',
    allowedHosts: 'all',
  },
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },
});
```

### webpack.prod.js

```javascript
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const { GenerateSW } = require('workbox-webpack-plugin');
const ImageMinimizerPlugin = require('image-minimizer-webpack-plugin');
const CriticalCssPlugin = require('critical-css-webpack-plugin');

module.exports = merge(common, {
  mode: 'production',
  devtool: 'source-map',
  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js',
  },
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
          },
        },
      }),
      new CssMinimizerPlugin(),
      new ImageMinimizerPlugin({
        minimizer: {
          implementation: ImageMinimizerPlugin.squooshMinify,
          options: {
            encodeOptions: {
              mozjpeg: { quality: 80 },
              webp: { lossless: 1 },
              avif: { cqLevel: 0 },
            },
          },
        },
      }),
    ],
    splitChunks: {
      chunks: 'all',
    },
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
      chunkFilename: '[id].[contenthash].css',
    }),
    new GenerateSW({
      clientsClaim: true,
      skipWaiting: true,
    }),
    new CriticalCssPlugin({
      base: 'dist/',
      src: 'index.html',
      target: 'index.html',
      extract: true,
      inline: true,
      minify: true,
      dimensions: [
        { width: 320, height: 568 },
        { width: 1200, height: 800 },
      ],
    }),
  ],
});
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES5",
    "module": "ESNext",
    "moduleResolution": "node",
    "jsx": "react",
    "jsxFactory": "h",
    "jsxFragmentFactory": "Fragment",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "lib": ["DOM", "ES2015", "ES2016", "ES2017"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@styles/*": ["src/styles/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.spec.ts"]
}
```

### .eslintrc.js

```javascript
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:preact/recommended',
    'plugin:jsx-a11y/recommended'
  ],
  plugins: ['preact', 'jsx-a11y'],
  settings: {
    preact: {
      version: 'detect'
    },
  },
  rules: {
    'preact/jsx-fragments': ['error', 'Fragment'],
  },
  env: {
    browser: true,
    node: true,
    es6: true
  },
  parserOptions: {
    ecmaVersion: 2020,
    sourceType: 'module',
    ecmaFeatures: {
      jsx: true
    }
  }
};
```

### package.json

```json
{
  "name": "my-preact-app",
  "version": "1.0.0",
  "description": "A fast and lightweight Preact application",
  "main": "src/index.js",
  "scripts": {
    "start": "webpack serve --config webpack/webpack.dev.js",
    "build": "webpack --config webpack/webpack.prod.js",
    "lint": "eslint src",
    "typecheck": "tsc --noEmit",
    "test": "jest",
    "analyze": "webpack --config webpack/webpack.analyze.js"
  },
  "keywords": ["preact", "webpack", "javascript"],
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {
    "preact": "^10.5.15",
    "preact-i18n": "^2.4.0",
    "goober": "^2.1.0"
  },
  "devDependencies": {
    "@babel/core": "^7.15.0",
    "@babel/plugin-proposal-class-properties": "^7.14.5",
    "@babel/plugin-syntax-dynamic-import": "^7.8.3",
    "@babel/plugin-transform-react-jsx": "^7.14.9",
    "@babel/preset-env": "^7.15.0",
    "@babel/preset-react": "^7.14.5",
    "@types/jest": "^27.0.1",
    "babel-loader": "^8.2.2",
    "clean-webpack-plugin": "^4.0.0",
    "copy-webpack-plugin": "^9.0.1",
    "critical-css-webpack-plugin": "^3.0.0",
    "css-loader": "^6.2.0",
    "css-minimizer-webpack-plugin": "^3.0.2",
    "dotenv-webpack": "^7.0.3",
    "eslint": "^7.32.0",
    "eslint-plugin-jsx-a11y": "^6.4.1",
    "eslint-plugin-preact": "^0.1.0",
    "eslint-webpack-plugin": "^3.0.1",
    "favicons-webpack-plugin": "^5.0.2",
    "html-webpack-plugin": "^5.3.2",
    "image-minimizer-webpack-plugin": "^2.2.0",
    "jest": "^27.0.6",
    "mini-css-extract-plugin": "^2.2.0",
    "postcss-loader": "^6.1.1",
    "responsive-loader": "^2.3.0",
    "sharp": "^0.29.0",
    "style-loader": "^3.2.1",
    "terser-webpack-plugin": "^5.1.4",
    "ts-jest": "^27.0.5",
    "ts-loader": "^9.2.5",
    "typescript": "^4.3.5",
    "webpack": "^5.51.1",
    "webpack-cli": "^4.8.0",
    "webpack-dev-server": "^4.0.0",
    "webpack-merge": "^5.8.0",
    "workbox-webpack-plugin": "^6.2.4"
  },
  "browserslist": [
    "last 2 versions",
    "not dead",
    "not < 2%"
  ]
}
```

## Setup Instructions

1. Create a new directory for your project and navigate into it:
   ```bash
   mkdir my-preact-app && cd my-preact-app
   ```

2. Initialize a new npm project:
   ```bash
   npm init -y
   ```

3. Replace the content of the generated `package.json` with the one provided above.

4. Install the dependencies:
   ```bash
   npm install
   ```

5. Create the configuration files as shown in the previous sections (`webpack.common.js`, `webpack.dev.js`, `webpack.prod.js`, `project.config.js`, `tsconfig.json`, `.eslintrc.js`).

6. Create your Preact application in the `src` directory. Start with an `index.js` file:

   ```javascript
   import { h, render } from 'preact';
   import App from './App';

   render(<App />, document.body);
   ```

7. Create an `App.js` file in the `src` directory:

   ```javascript
   import { h } from 'preact';

   export default function App() {
     return <h1>Hello, Preact!</h1>;
   }
   ```

8. Create a `public` directory with an `index.html` file:

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>My Preact App</title>
   </head>
   <body>
   </body>
   </html>
   ```

9. Run `npm start` to start the development server or `npm run build` to create a production build.

## API

The `project.config.js` file serves as the main configuration API. You can adjust the following options:

- `projectName`: Name of your project
- `projectDescription`: Brief description of your project
- `srcDir`: Source directory
- `publicDir`: Public assets directory
- `outputDir`: Build output directory
- `entry`: Entry point of your application
- `htmlTemplate`: HTML template file
- `favicon`: Favicon file
- `devServerPort`: Development server port
- `devServerProxy`: Proxy configuration for the dev server
- `aliases`: Path aliases for easier imports
- `images`: Configuration for image processing
  - `sizes`: Array of image sizes to generate
  - `placeholderSize`: Size of the placeholder image
  - `outputPath`: Output path for processed images
  - `quality`: Quality of the generated images

## FAQ

1. **Q: Can I use this configuration with React instead of Preact?**
   A: While this configuration is optimized for Preact, you can adapt it for React by removing Preact-specific aliases and adjusting the Babel configuration. You'll need to update the `resolve.alias` in `webpack.common.js` and modify the Babel presets and plugins accordingly.

2. **Q: How do I add support for Sass or Less?**
   A: To add support for Sass or Less, install the appropriate loaders (sass-loader or less-loader) and add them to the CSS rule in `webpack.common.js`. For example, for Sass:
   ```bash
   npm install --save-dev sass sass-loader
   ```
   Then update the CSS rule in `webpack.common.js`:
   ```javascript
   {
     test: /\.(css|scss)$/,
     use: ['style-loader', 'css-loader', 'postcss-loader', 'sass-loader'],
   }
   ```

3. **Q: Can I use this configuration for a TypeScript project?**
   A: Yes, this configuration already includes TypeScript support. Just use `.ts` or `.tsx` file extensions for your TypeScript files. The `tsconfig.json` file is already set up to handle TypeScript compilation.

4. **Q: How do I add new environment variables?**
   A: Add new environment variables to your `.env.development` or `.env.production` files. They will be automatically included in your build thanks to the `Dotenv` plugin in the webpack configuration. You can then access these variables in your code as `process.env.YOUR_VARIABLE_NAME`.

5. **Q: How can I customize the service worker generation?**
   A: To customize the service worker generation, modify the `GenerateSW` plugin configuration in `webpack.prod.js`. You can adjust options like `clientsClaim`, `skipWaiting`, or add custom runtime caching rules. Refer to the Workbox documentation for more advanced configurations.

## Conclusion

This webpack configuration provides a robust foundation for building Preact applications. It offers a balance of performance optimizations, developer experience improvements, and modern web development features. The setup is flexible and can be customized to fit the specific needs of your project.

Remember to regularly update your dependencies and adjust the configuration as new best practices and tools emerge in the fast-evolving JavaScript ecosystem.

Happy coding!
