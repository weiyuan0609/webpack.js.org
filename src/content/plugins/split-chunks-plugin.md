---
title: SplitChunksPlugin
contributors:
  - sokra
  - jeremenichelli
  - Priestch
  - chrisdothtml
  - EugeneHlushko
  - byzyk
  - jacobangel
  - madhavarshney
  - sakhisheikh
  - superburrito
  - ryandrew14
  - snitin315
  - chenxsan
  - rohrlaf
related:
  - title: webpack's automatic deduplication algorithm example
    url: https://github.com/webpack/webpack/blob/master/examples/many-pages/README.md
  - title: "webpack 4: Code Splitting, chunk graph and the splitChunks optimization"
    url: https://medium.com/webpack/webpack-4-code-splitting-chunk-graph-and-the-splitchunks-optimization-be739a861366
---

最初，chunks（以及内部导入的模块）是通过内部 webpack 图谱中的父子关系关联的。`CommonsChunkPlugin` 曾被用来避免他们之间的重复依赖，但是进一步的优化是不可能的。

从 webpack v4 开始，移除了 `CommonsChunkPlugin`，取而代之的是 `optimization.splitChunks`。


## 默认值 {#defaults}

对于大部分用户来说，`SplitChunksPlugin` 是开箱即用的，而且很适用。

默认情况下，它只会影响到按需加载的 chunks，因为修改 initial chunks 会影响到项目的 HTML 文件中的脚本标签。

webpack 将根据以下条件自动拆分 chunks：

- 新的 chunk 可以被多个 chunk 分享，或者它来自于 node_modules 文件夹
- 新的 chunk 体积大于 20kb（在进行 min+gz 之前的体积）
- 当按需加载 chunks 时，并行请求的最大数量小于或等于 30
- 当加载初始化页面时，并发请求的最大数量小于或等于 30

当尝试满足最后两个条件时，最好使用较大的 chunks。

## 配置 {#configuration}

webpack 为希望对该功能进行更多控制的开发人员提供了一组选项。

W> 选择了默认配置以适合 Web 性能最佳实践，但是项目的最佳策略可能有所不同。如果要更改配置，则应评估所做更改的影响，以确保有真正的收益。

## `optimization.splitChunks` {#optimizationsplitchunks}

下面这个配置对象代表 `SplitChunksPlugin` 的默认行为。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'all',
      minSize: 20000,
      minRemainingSize: 0,
      maxSize: 0,
      minChunks: 1,
      maxAsyncRequests: 30,
      maxInitialRequests: 30,
      automaticNameDelimiter: '~',
      enforceSizeThreshold: 50000,
      cacheGroups: {
        defaultVendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

W> 当 webpack 处理文件路径时，它们始终包含 `/` 在 Unix 系统和 `\` 在 Windows 上。这就是为什么 `[\\/]` 在 `{cacheGroup}.test` 字段中使用 in 来表示路径分隔符的原因。`/` 或 `\` in `{cacheGroup}.test` 会在跨平台使用时引起问题。

W> 从 webpack 5 开始，不再允许将条目名称传递给 `{cacheGroup}.test` 现有的 chunk 并使用现有的 chunk 的名称 `{cacheGroup}.name`。

### `splitChunks.automaticNameDelimiter` {#splitchunksautomaticnamedelimiter}

`string = '~'`

默认情况下，webpack 将使用 chunk 的来源和名称生成名称（例如 `vendors~main.js`）。此选项使您可以指定用于生成名称的定界符。

### `splitChunks.chunks` {#splitchunkschunks}

`string = 'all'` `function (chunk)`

这表明将选择哪些 chunk 进行优化。当提供一个字符串，有效值为 `all`，`async` 和 `initial`。提供 `all` 可能特别强大，因为这意味着即使在异步和非异步 chunks 之间也可以共享 chunks。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // include all types of chunks
      chunks: 'all'
    }
  }
};
```

或者，您也可以提供一个函数去做更多的控制。这个函数的返回值将决定是否包含每一个 chunk。

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks (chunk) {
        // exclude `my-excluded-chunk`
        return chunk.name !== 'my-excluded-chunk';
      }
    }
  }
};
```

T> 您可以将此配置与 [HtmlWebpackPlugin](/plugins/html-webpack-plugin/) 结合使用。它将为您注入所有生成的 vendor chunks。

### `splitChunks.maxAsyncRequests` {#splitchunksmaxasyncrequests}

`number = 30`

按需加载时的最大并行请求数。

### `splitChunks.maxInitialRequests` {#splitchunksmaxinitialrequests}

`number = 30`

入口点的最大并行请求数。

### `splitChunks.minChunks` {#splitchunksminchunks}

`number = 1`

拆分前必须共享模块的最小 chunks 数。

### `splitChunks.minSize` {#splitchunksminsize}

`number = 20000`

生成 chunk 的最小体积（以 bytes 为单位）。

### `splitChunks.enforceSizeThreshold` {#splitchunksenforcesizethreshold}

#### `splitChunks.cacheGroups.{cacheGroup}.enforceSizeThreshold` {#splitchunkscachegroupscachegroupenforcesizethreshold}

`number = 50000`

强制执行拆分的体积阈值和其他限制（minRemainingSize，maxAsyncRequests，maxInitialRequests）将被忽略。


### `splitChunks.minRemainingSize` {#splitchunksminremainingsize}

#### `splitChunks.cacheGroups.{cacheGroup}.minRemainingSize` {#splitchunkscachegroupscachegroupminremainingsize}

`number = 0`

在 webpack 5中引入了 `splitChunks.minRemainingSize` 选项，通过确保拆分后剩余的最小 chunk 体积超过限制来避免大小为零的模块。 默认为 ['development' mode](/configuration/mode/#mode-development) 中的 `0`。对于其他情况，`splitChunks.minRemainingSize` 默认为 `splitChunks.minSize` 的值，因此除需要深度控制的极少数情况外，不需要手动指定它。

W> `splitChunks.minRemainingSize` 仅在剩余单个 chunk 时生效。

### `splitChunks.maxSize` {#splitchunksmaxsize}

`number = 0`

使用 `maxSize`（每个缓存组 `optimization.splitChunks.cacheGroups [x] .maxSize` 全局使用 `optimization.splitChunks.maxSize` 或对后备缓存组 `optimization.splitChunks.fallbackCacheGroup.maxSize` 使用）告诉 webpack 尝试尝试将大于 `maxSize` 个字节的 chunks 分割成较小的部分。 这些较小的部分在体积上至少为 `minSize`（仅次于 `maxSize`）。
该算法是确定性的，对模块的更改只会产生局部影响。 这样，在使用长期缓存时就可以使用它并且不需要记录。 `maxSize` 只是一个提示，当模块大于 `maxSize` 时可能会被违反，否则拆分会违反 `minSize`。

当 chunk 已经有一个名称时，每个部分将获得一个从该名称派生的新名称。 根据 `optimization.splitChunks.hidePathInfo` 的值，它将添加一个从第一个模块名称或其哈希值派生的密钥。

`maxSize` 选项旨在与 HTTP/2 和长期缓存一起使用。它增加了请求数量以实现更好的缓存。它还可以用于减小文件大小，以加快重建速度。

T> `maxSize` 比 `maxInitialRequest/maxAsyncRequests` 具有更高的优先级。实际优先级是 `maxInitialRequest/ maxAsyncRequests < maxSize < minSize`。

T> 设置 `maxSize` 的值会同时设置 `maxAsyncSize` 和 `maxInitialSize` 的值。

### `splitChunks.maxAsyncSize` {#splitchunksmaxasyncsize}

`number`

像 `maxSize` 一样，`maxAsyncSize` 可以全局应用（`splitChunks.maxAsyncSize`），cacheGroups（`splitChunks.cacheGroups.{cacheGroup}.maxAsyncSize`）或后备缓存组（`splitChunks.fallbackCacheGroup.maxAsyncSize` ）。

`maxAsyncSize` 和 `maxSize` 的区别在于 `maxAsyncSize` 仅会影响按需加载 chunks。

### `splitChunks.maxInitialSize` {#splitchunksmaxinitialsize}

`number`

像 `maxSize` 一样，`maxInitialSize` 可以全局应用（splitChunks.maxInitialSize），cacheGroups（`splitChunks.cacheGroups.{cacheGroup}.maxInitialSize`）或后备缓存组（`splitChunks.fallbackCacheGroup.maxInitialSize`）。

`maxInitialSize` 和 `maxSize` 的区别在于 `maxInitialSize` 仅会影响初始加载 chunks。

### `splitChunks.name` {#splitchunksname}

`boolean = true` `function (module, chunks, cacheGroupKey) => string` `string`

每个 cacheGroup 也可以使用： `splitChunks.cacheGroups.{cacheGroup}.name`。

拆分 chunk 的名称。 提供 `true` 将基于 chunks 和缓存组密钥自动生成一个名称。

提供字符串或函数使您可以使用自定义名称。指定字符串或始终返回相同字符串的函数会将所有常见模块和 vendor 合并为一个 chunk。这可能会导致更大的初始下载量并减慢页面加载速度。

如果您选择指定一个函数，则可能会发现 `chunk.name` 和 `chunk.hash` 属性（其中 `chunk` 是 `chunks` 数组的一个元素）在选择 chunk 名时特别有用。

如果 `splitChunks.name` 与 [entry point](/configuration/entry-context/#entry) 名称匹配，entry point 将被删除。

T> 对于生产版本，建议将 `splitChunks.name` 设置为 `false`，以免不必要地更改名称。

__main.js__

```js
import _ from 'lodash';

console.log(_.join(['Hello', 'webpack'], ' '));
```

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        commons: {
          test: /[\\/]node_modules[\\/]/,
          // cacheGroupKey here is `commons` as the key of the cacheGroup
          name(module, chunks, cacheGroupKey) {
            const moduleFileName = module.identifier().split('/').reduceRight(item => item);
            const allChunksNames = chunks.map((item) => item.name).join('~');
            return `${cacheGroupKey}-${allChunksNames}-${moduleFileName}`;
          },
          chunks: 'all'
        }
      }
    }
  }
};
```

使用以下 `splitChunks` 配置来运行 webpack 也会输出一组公用组，其下一个名称为：`commons-main-lodash.js.e7519d2bb8777058fa27.js`（以散列方式作为真实世界输出示例）。

W> 在为不同的拆分 chunk 分配相同的名称时，所有 vendor 模块都放在一个共享的 chunk 中，尽管不建议这样做，因为这可能会导致下载更多代码。

### `splitChunks.automaticNamePrefix` {#splitchunksautomaticnameprefix}

`string = ''`

为创建的 chunks 设置名称前缀。

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      automaticNamePrefix: 'general-prefix',
      cacheGroups: {
        react: {
          // ...
          automaticNamePrefix: 'react-chunks-prefix'
        }
      }
    }
  }
};
```

### `splitChunks.cacheGroups` {#splitchunkscachegroups}

缓存组可以继承和 / 或覆盖来自 `splitChunks。*` 的任何选项。但是 `test`，`priority` 和 `reuseExistingChunk` 只能在缓存组级别上进行配置。要禁用任何默认缓存组，请将它们设置为 `false`。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        default: false
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.{cacheGroup}.priority` {#splitchunkscachegroupscachegrouppriority}

`number`

一个模块可以属于多个缓存组。优化将优先考虑具有更高 `priority` 的缓存组 默认组的优先级为负，以允许自定义组获得更高的优先级（自定义组的默认值为 `0`）。

#### `splitChunks.cacheGroups.{cacheGroup}.reuseExistingChunk` {#splitchunkscachegroupscachegroupreuseexistingchunk}

`boolean`

如果当前 chunk 包含已从主 bundle 中拆分出的模块，则它将被重用，而不是生成新的模块。 这可能会影响 chunk 的结果文件名。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.{cacheGroup}.type` {#splitchunkscachegroupscachegrouptype}

`function` `RegExp` `string`

允许按模块类型将模块分配给缓存组。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        json: {
          type: 'json'
        }
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.test` {#splitchunkscachegroupstest}

#### `splitChunks.cacheGroups.{cacheGroup}.test` {#splitchunkscachegroupscachegrouptest}

`function (module, chunk) => boolean` `RegExp` `string`

控制此缓存组选择的模块。省略它会选择所有模块。它可以匹配绝对模块资源路径或 chunk 名称。匹配 chunk 名称时，将选择 chunk 中的所有模块。

为 `{cacheGroup}.test` 提供一个功能：

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        svgGroup: {
          test(module, chunks) {
            // `module.resource` contains the absolute path of the file on disk.
            // Note the usage of `path.sep` instead of / or \, for cross-platform compatibility.
            const path = require('path');
            return module.resource &&
                 module.resource.endsWith('.svg') &&
                 module.resource.includes(`${path.sep}cacheable_svgs${path.sep}`);
          }
        },
        byModuleTypeGroup: {
          test(module, chunks) {
            return module.type === 'javascript/auto';
          }
        }
      }
    }
  }
};
```

为了查看 `module` and `chunks` 对象中可用的信息，您可以在回调函数中放入 `debugger;` 语句。然后 [以调试模式运行 webpack 构建](/contribute/debugging/#devtools) 检查 Chromium DevTools 中的参数。

向 `{cacheGroup}.test` 提供 `RegExp`：

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          // Note the usage of `[\\/]` as a path separator for cross-platform compatibility.
          test: /[\\/]node_modules[\\/]|vendor[\\/]analytics_provider|vendor[\\/]other_lib/
        }
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.{cacheGroup}.filename` {#splitchunkscachegroupscachegroupfilename}

`string` `function (pathData, assetInfo) => string`

仅在初始 chunk 时才允许覆盖文件名。
也可以在 [`output.filename`](/configuration/output/#outputfilename) 中使用所有占位符。

W> 也可以在 `splitChunks.filename` 中全局设置此选项，但是不建议这样做，如果 [`splitChunks.chunks`](#splitchunkschunks) 未设置为 `'initial'`，则可能会导致错误。避免全局设置。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          filename: '[name].bundle.js'
        }
      }
    }
  }
};
```

若为函数，则：

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          filename: (pathData) => {
            // Use pathData object for generating filename string based on your requirements
            return `${pathData.chunk.name}-bundle.js`;
          }
        }
      }
    }
  }
};
```

通过提供以文件名开头的路径 `'js/vendor/bundle.js'`，可以创建文件夹结构。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          filename: 'js/[name]/bundle.js'
        }
      }
    }
  }
};
```


#### `splitChunks.cacheGroups.{cacheGroup}.enforce` {#splitchunkscachegroupscachegroupenforce}

`boolean = false`

告诉 webpack 忽略 [`splitChunks.minSize`](#splitchunksminsize)，[`splitChunks.minChunks`](#splitchunksminchunks)，[`splitChunks.maxAsyncRequests`](#splitchunksmaxasyncrequests) 和 [`splitChunks.maxInitialRequests`](#splitchunksmaxinitialrequests) 选项，并始终为此缓存组创建 chunks。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          enforce: true
        }
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.{cacheGroup}.idHint` {#splitchunkscachegroupscachegroupidhint}

`string`

设置 chunk id 的提示。 它将被添加到 chunk 的文件名中。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        defaultVendors: {
          idHint: 'vendors'
        }
      }
    }
  }
};
```

## Examples {#examples}

### Defaults: Example 1 {#defaults-example-1}

```js
// index.js

import('./a'); // dynamic import
```

```js
// a.js
import 'react';

//...
```

__结果：__ 将创建一个单独的包含 `react` 的 chunk。在导入调用中，此 chunk 并行加载到包含 `./a` 的原始chunk中。

为什么？

- 条件1：chunk 包含来自 `node_modules` 的模块
- 条件2：`react` 大于 30kb
- 条件3：导入调用中的并行请求数为 2
- 条件4：在初始页面加载时不影响请求

这背后的原因是什么？`react` 可能不会像您的应用程序代码那样频繁地更改。通过将其移动到单独的 chunk 中，可以将该 chunk 与应用程序代码分开进行缓存（假设您使用的是 chunkhash，records，Cache-Control 或其他长期缓存方法）。

### Defaults: Example 2 {#defaults-example-2}

```js
// entry.js

// dynamic imports
import('./a');
import('./b');
```

```js
// a.js
import './helpers'; // helpers is 40kb in size

//...
```

```js
// b.js
import './helpers';
import './more-helpers'; // more-helpers is also 40kb in size

//...
```

__结果：__ 将创建一个单独的 chunk，其中包含 `./helpers` 及其所有依赖项。在导入调用时，此 chunk 与原始 chunks 并行加载。

为什么？

- 条件1：chunk 在两个导入调用之间共享
- 条件2：`helpers` 大于 30kb
- 条件3：导入调用中的并行请求数为 2
- 条件4：在初始页面加载时不影响请求

将 `helpers` 的内容放入每个 chunk 中将导致其代码被下载两次。通过使用单独的块，这只会发生一次。我们会支付额外请求的费用，这可以视为一种折衷。这就是为什么最小体积为 30kb 的原因。

### Split Chunks: Example 1 {#split-chunks-example-1}

创建一个 `commons` chunk，其中包括 entry points 之间共享的所有代码。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        commons: {
          name: 'commons',
          chunks: 'initial',
          minChunks: 2
        }
      }
    }
  }
};
```

W> 此配置可以扩大您的初始 bundles，建议在不需要立即使用模块时使用动态导入。

### Split Chunks: Example 2 {#split-chunks-example-2}

创建一个 `vendors` chunk，其中包括整个应用程序中 `node_modules` 中的所有代码。

__webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        commons: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

W> 这可能会导致包含所有外部程序包的较大 chunk。建议仅包括您的核心框架和实用程序，并动态加载其余依赖项。

### Split Chunks: Example 3 {#split-chunks-example-3}

 创建一个 `custom vendor` chunk，其中包含与 `RegExp` 匹配的某些 `node_modules` packages。

 __webpack.config.js__

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'vendor',
          chunks: 'all',
        }
      }
    }
  }
};
```

T> 这将导致将  `react`和 `react-dom` 分成一个单独的 chunk。 如果您不确定 chunk 中包含哪些 packages，请参考 [Bundle Analysis](/guides/code-splitting/#bundle-analysis) 部分以获取详细信息。
