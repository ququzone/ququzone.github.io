---
layout: post
title: ckb-cache-js 可嵌入的Live Cell Cache库
---

由于Nervos CKB的Cell模式是类似UTXO的模型，那么在组装交易时需要明确Inputs的输入，这样不管是开发钱包、dApp Server还是其他任何需要和CKB交互的应用场景，都需要一个查询live cells的功能。尽管CKB RPC已经提供了基于LockHash查询live cells的功能，这个这个功能是通用型的功能，并且只能根据LockHash查询，在很多dApp场景中无法适用。

目前正是CKB主链dApp开发的起始阶段，很多应用都需要灵活的查询live cells，例如LockHash、TypeHash、CodeHash等等查询Cells，基于这个场景，我开发了[ckb-cache-js](https://github.com/ququzone/ckb-cache-js)。考虑到很多dApp的开发技术栈是JavaScript或者TypeScript(https://www.typescriptlang.org/)，正如ckb-cache-js的命名一样，这个cache库是基于TypeScript语言开发的。

### ckb-cache-js 技术栈

ckb-cache-js 数据存储采用 SQLite3（将来有可能改为性能更高的LevelDB），并且ckb-cache-js可以嵌入到任何基于JavaScript/TypeScript开发的应用中，为该应用提供cell的cache层功能。ckb-cache-js的定位是提供有限规则的live cell缓存，并不适合存储全量的live cell。

### 安装

ckb-cache-js 已经发布到npm中心仓库中，可以通过下述命令安装:

```
npm -i ckb-cache-js -S
```

### 使用

1. 启动

```
const CKB = require("@nervosnetwork/ckb-sdk-core").default;
const { DefaultCacheService, initConnection } = require("ckb-cache-js");

let cache;
const start = async (nodeUrl = "http://localhost:8114") => {
  await initConnection({
    "type": "sqlite",
    "database": "database.sqlite",
    "synchronize": true,
    "logging": false,
    "entities": [
      "node_modules/ckb-cache-js/lib/database/entity/*.js"
    ]
  });

  const ckb = new CKB(nodeUrl);
  cache = new DefaultCacheService(ckb);
  cache.start();
};
start();
```

2. 添加缓存规则

```
await addRule({name: "LockHash": "0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26"}, "1000");
```

上述命令是添加一条 `LockHash` 等于 `0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26` 的缓存规则，并且从区块 `1000` 开始重新扫描，所有满足该规则的live cells将存储到 `cell` 数据库表中。目前支持的规则如下:

- LockHash
- LockCodeHash
- TypeHash
- TypeCodeHash
- Data

3. 查询

```
const BN = require("bn.js");
const { QueryBuilder } = require("ckb-cache-js");

// query by lockhash
const allByLockhash = await cache.findCells(
  QueryBuilder.create()
    .setLockHash(account.lock)
    .build()
);

// query by capacity
const byCapacity = await cache.findCells(
  QueryBuilder.create()
    .setLockHash("0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26")
    .setTypeCodeHash("null")
    .setData("0x")
    .setCapacity("10000000000")
    .build()
);

// query by udt
const byUdt = await cache.findCells(
  QueryBuilder.create()
    .setLockHash("0x6a242b57227484e904b4e08ba96f19a623c367dcbd18675ec6f2a71a0ff4ec26")
    .setTypeHash("0xcc77c4deac05d68ab5b26828f0bf4565a8d73113d7bb7e92b8362b8a74e58e58")
    .setCapacityFetcher((cell: Cell) => {
      return new BN(Buffer.from(cell.data.slice(2), "hex"), 16, "le")
    })
    .build()
);
```

### 协助

目前 [ckb-cache-js](https://github.com/ququzone/ckb-cache-js) 尚在开发阶段，还有需要尚待改进的点，欢迎提交问题或者贡献代码。
