## 基础

```
1. A->B seq:0(177947175)  ack:1              SYN
2. B->A seq:0(1183805901) ack:1(177947176)   SYN ACK
3. A->B seq:1(177947176)  ack:1(1183805902)  ACK

4. A->B seq:1(177947176)  ack:1(1183805902)  PSH ACK  data(40bytes)  OPEN
5. B->A seq:1(1183805902) ack:41(177947216)  ACK

6. B->A seq:1(1183805902) ack:41(177947216)  PSH ACK  data(48bytes)  OPEN
7. A->B seq:41(177947216) ack:49(1183805950) ACK

8. A->B seq:41(177947216)  ack:49(1183805950) PSH ACK  data(4bytes) keepalive ---第11个报文被确认
9. B->A seq:49(1183805950) ack:41(177947216)  PSH ACK  data(4bytes) keepalive
10.A->B seq:45(177947216)  ack:53(1183805954) ACK
11.B->A seq:53(1183805954) ack:45(177947220)  ACK ---确认的第8个报文

12. A->B seq:45(177947220)  ack:53(1183805954) FIN ACK
13. B->A seq:53(1183805954) ack:46(177947221)  ACK
14. B->A seq:53(1183805954) ack:46(177947221)  FIN ACK
15. A->B seq:45(177947221)  ack:54(1183805955) ACK
```

> 在三次握手后，实际数据传输时，seq就是对端发送过来的ack，ack就是就是对端发送过来的seq+data

参考: [TCP包的seq和ack号计算方法](https://blog.csdn.net/huaishu/article/details/93739446)


