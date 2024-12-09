# 通信ACK机制实现
## 1.非阻塞形式
不影响其余收发通信；但非常占用资源；待优化
1. 消息待发送
2. event/mq发送；
3. 线程recv event/mq
    ACK处理线程：
    a. 阻塞等待event；      
    b. copy存储msg_send;一轮发送；起soft_timer（循环）；flag置1；   
    c. timer超时重复发送msg_send；counter++；   
    d. uart_tx线程异步等待，接收/解析ACK；  
    e. 对应timer stop；copy/flag 清0；  

## 2.非阻塞形式
1. 消息发送（当前FAN/IPD/HERATBEAT）
2. wait ACK；macro TMEOUT 300ms；
3. 超时重发；
4. 最合理做法：TX/RX拆分
5. 发送器msg copy; flag_ACK=0; send event；
6. TX_thread发送后阻塞等待RX解析到ACK；flag_ACK=1; end; flag_ACK=0; 