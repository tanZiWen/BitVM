<h2>hoststuff原生共识</h2>

机器配置：MacBook, 芯片M1, 内存16GB

1. 签名算法： ECDSA
2. 账号模型: balance模型
3. 共识节点个数：4
4. 以太坊交易size：200b左右


| 交易size(B) | TPS(tx/s) | 交易Batch |
| ----------- | --------- | --------- |
| 256         | 17,029    | 1         |
| 256         | 284,644   | 15_000    |
| 1024        | 10,599    | 1         |
| 1024        | 41,349    | 15_000    |
| 132*1024    | 69        | 1         |
| 132*1024    | 205       | 15_000    |


<h2>TEST1</h2>

1. 交易size: 1kb
2. 最大TPS: 121,980 tx/s
3. batch_size: 15_000
4. 延迟: 11 ms
<img width="431" height="391" alt="image" src="https://github.com/user-attachments/assets/957b08ca-898c-4c55-88e7-644264d7fbc0" />

<h2>TEST2</h2>

1. 交易size: 132kb
2. 最大tps: 513 tx/s
3. 延迟：49/ms
4. batch_size: 15_000
<img width="845" height="521" alt="image" src="https://github.com/user-attachments/assets/46243c69-d324-4d46-b3e8-b71fc6f8f115" />

<H2>TEST3</H2>

1. 交易size: 132kb
2. 最大tps: 341 tx/s
3. 延迟：350 ms
4. batch_size: 1

<img width="578" height="485" alt="image" src="https://github.com/user-attachments/assets/d67791b4-df97-48e9-b92d-4d3f9f6ebdda" />

<h2>Prove Time</h2>
<img width="781" height="202" alt="image" src="https://github.com/user-attachments/assets/6cfe5401-878e-4ed2-9e8f-0c86593af322" />

