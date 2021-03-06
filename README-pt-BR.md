# Web-pack Guia Inicial

## Objetivo deste guia

Este é um guia de como fazer as "coisas" utilizando o webpack. Este guia inclui a maioria das coisas que utilizamos no Instagram e "nada que não utilizamos".

Meu conselho: comece com este guia como se fosse a documentação do webpack, e então apenas utilize a documentação oficial para esclarecimento.

## Pré-requisitos

  * Conhecer browserify, RequireJS ou alguma ferramenta similar
  * Ver o valor de:
    * Divisão de pacotes
    * Carregamento Assícrono
    * Empacotamento de recursos estáticos como imagens e CSS

## 1. Por que webpack?


  * **É como browserify** mas pode dividir sua aplicação em vários arquivos. Se você tem várias páginas em sua aplicação SPA (Single Page App), o usuário baixa o código apenas para aquela página. Se ele vai para outra página, ele não irá baixar novamente código em comum.

  * **Frequentemente substitui grunt ou gulp** porque pode gerar e empacotar CSS, CSS pré-processado, linguagens que compilam para JS e imagens, entre outras coisas.

Suporta AMD e CommonJS, entre outros sistemas modulares (Angular, ES6). Se você não sabe o que usar, use CommonJS.

## 2. Webpack pra quem usa Browserify

Esses são equivalentes:

```js
browserify main.js > bundle.js
```

```js
webpack main.js bundle.js
```

Porém, webpack é mais poderoso que o Browserify, sendo assim você geralmente irá querer criar um arquivo `webpack.config.js` pra manter as coisas organizadas:

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  }
};
```

Isso é só JS, então sinta-se livre pra colocar código de verdade lá.

## 3. Como executar o webpack

Mude para o diretório contendo `webpack.config.js` e execute:

  * `webpack` pra construir uma vez para o desenvolvimento
  * `webpack -p` pra construir uma vez para produção (minificação)
  * `webpack --watch` pra construir continuamente de forma incremental no desenvolvimento (rápido!)
  * `webpack -d` para incluir source maps

## 4. Linguagens que compilam para JS

O equivalente do webpack para as transformações do browserify e plugins do RequireJS é um **carregador (loader)**. Aqui está como você pode ensinar o webpack a carregar o suporte para o CoffeeScript e Facebook JSX+ES6 (você deve executar `npm install babel-loader coffee-loader`):

Veja também [instruções de instalação do babel-loader](https://www.npmjs.com/package/babel-loader) para dependências adicionais (tl;dr execute `npm install babel-core babel-preset-es2015 babel-preset-react`).

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  },
  module: {
    loaders: [
      { test: /\.coffee$/, loader: 'coffee-loader' },
      {
        test: /\.js$/,
        loader: 'babel-loader',
        query: {
          presets: ['es2015', 'react']
        }
      }
    ]
  }
};
```

Para habilitar exigindo arquivos sem especificar a extensão, você deve adicionar um parâmetro `resolve.extensions` especificando quais arquivos o webpack deve procurar:

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  },
  module: {
    loaders: [
      { test: /\.coffee$/, loader: 'coffee-loader' },
      {
        test: /\.js$/,
        loader: 'babel-loader',
        query: {
          presets: ['es2015', 'react']
        }
      }
    ]
  },
  resolve: {
    // agora você pode usar require('file') ao invés de require('file.coffee')
    extensions: ['', '.js', '.json', '.coffee']
  }
};
```


## 5. Folhas de estilos e imagens

Primeiro atualize o seu código para importar seus arquivos estáticos usando `require()` (nomeando como é feito com o `require()` do Node):

```js
require('./bootstrap.css');
require('./myapp.less');

var img = document.createElement('img');
img.src = require('./glyph.png');
```

Quando você importa um CSS (ou less, etc), o webpack coloca o CSS como uma string em linha, dentro do pacote Javascript e a função `require()` irá inserir uma tag `<style>` na página. Quando você importar imagens, o webpack vai colocar em linha a URL para a imagem dentro do pacote e retornando a URL ao chamar a função `require()`.

Mas você precisa ensinar o webpack a fazer isso (novamente, com **carregadores(loaders)**):

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    path: './build', // Isso é onde as imagens e o javascript vão
    publicPath: 'http://mycdn.com/', // Isso é utilizado para gerar URLs para, por ex. imagens
    filename: 'bundle.js'
  },
  module: {
    loaders: [
      { test: /\.less$/, loader: 'style-loader!css-loader!less-loader' }, // use ! para encadear carregadores(loaders)
      { test: /\.css$/, loader: 'style-loader!css-loader' },
      { test: /\.(png|jpg)$/, loader: 'url-loader?limit=8192' } // coloca em linha a URLs em formato base64 para imagens <= 8k, coloca URLs diretas para o resto
    ]
  }
};
```

## 6. Funcionalidades de flags

Nós tempos códigos que queremos abrir somente para nosso ambiente de desenvolvimento (como logging) e nossos próprios servidores internos (como funcionalidades não-oficiais que estamos testando com os colaboradores). No nosso código, referimos para globais mágicas:

```js
if (__DEV__) {
  console.warn('Extra logging');
}
// ...
if (__PRERELEASE__) {
  showSecretFeature();
}
```

Então mostre para o webpack essas globais mágicas:

```js
// webpack.config.js

// definePlugin aceita strings puras e insere elas. Então, você pode colocar strings do JS se quiser.
var definePlugin = new webpack.DefinePlugin({
  __DEV__: JSON.stringify(JSON.parse(process.env.BUILD_DEV || 'true')),
  __PRERELEASE__: JSON.stringify(JSON.parse(process.env.BUILD_PRERELEASE || 'false'))
});

module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  },
  plugins: [definePlugin]
};
```

Agora você pode fazer o build com `BUILD_DEV=1 BUILD_PRERELEASE=1 webpack` pelo terminal. Lembre-se de que `webpack -p` executa obfuscação de eliminação de código-morto, qualquer coisa dentro destes blocos serão removidos, então você não terá suas strings ou funcionalidades secretas reveladas.

## 7. Múltiplos pontos de entrada

Vamos dizer que você tem uma página de perfil e uma página de feed. Você não quer fazer o usuário baixar o código para a página de feed se ele só quer a página de perfil. Então faça múltiplos pacotes: crie um "módulo principal" (chamado como ponto de entrada) por página:

```js
// webpack.config.js
module.exports = {
  entry: {
    Profile: './profile.js',
    Feed: './feed.js'
  },
  output: {
    path: 'build',
    filename: '[name].js' // Template baseado nas chaves de entrada acima
  }
};
```

Para o perfil, insira `<script src="build/Profile.js"></script>` na página. Siga o mesmo raciocínio para a página de feed.

## 8. Optimizing common code

Feed and Profile share a lot in common (like React and the common stylesheets and components). webpack can figure out what they have in common and make a shared bundle that can be cached between pages:

```js
// webpack.config.js

var webpack = require('webpack');

var commonsPlugin =
  new webpack.optimize.CommonsChunkPlugin('common.js');

module.exports = {
  entry: {
    Profile: './profile.js',
    Feed: './feed.js'
  },
  output: {
    path: 'build',
    filename: '[name].js' // Template based on keys in entry above
  },
  plugins: [commonsPlugin]
};
```

Add `<script src="build/common.js"></script>` before the script tag you added in the previous step and enjoy the free caching.

## 9. Async loading

CommonJS is synchronous but webpack provides a way to asynchronously specify dependencies. This is useful for client-side routers, where you want the router on every page, but you don't want to have to download features until you actually need them.

Specify the **split point** where you want to load asynchronously. For example:

```js
if (window.location.pathname === '/feed') {
  showLoadingState();
  require.ensure([], function() { // this syntax is weird but it works
    hideLoadingState();
    require('./feed').show(); // when this function is called, the module is guaranteed to be synchronously available.
  });
} else if (window.location.pathname === '/profile') {
  showLoadingState();
  require.ensure([], function() {
    hideLoadingState();
    require('./profile').show();
  });
}
```

webpack will do the rest and generate extra **chunk** files and load them for you.

webpack will assume that those files are in your root directory when you load then into a html script tag for example. You can use `output.publicPath` to configure that.

```js
// webpack.config.js
output: {
    path: "/home/proj/public/assets", //path to where webpack will build your stuff
    publicPath: "/assets/" //path that will be considered when requiring your files
}
```

## Recursos Adicionais

Dê uma olhada em um exemplo real de como uma equipe bem sucedida está alavancando o webpack: http://youtu.be/VkTCL6Nqm6Y
Este é Pete Hunt na OSCon falando sobre o uso de webpack no Instagram.com

## FAQ

### webpack não parece modular

webpack é **extremamente** modular. O que faz o webpack excelente é que ele deixa plugins se injetarem em mais lugares no processo de construção quando comparado à alternativas como browserify e requirejs. Muitas coisas que podem parecer incorporadas ao `core` são apenas plugins que são carregados por padrão e podem ser sobrescritos (ex: o parser require() do CommonJS).
