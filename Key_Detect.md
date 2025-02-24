# Key Detetc 逻辑汇总

## 完全事件/中断驱动-无线程轮询
IDLE状态下不占资源；但是**逻辑复杂+难支持多按键**复用

1. 中断+Flga-->驱动事件：
```C
void key_irq(void *arg)
{
    if (key_event && clicked_flag == 0 && wait_flag == 0) {
        rt_event_send(key_event, EVENT_FLAG_FIRST_CLICK);
    } else if (key_event && clicked_flag == 1 && wait_flag == 0) {
        rt_event_send(key_event, EVENT_FLAG_SECOND_CLICK);
    }
}
```

2.事件(实际替代状态机角色)-->驱动状态转换逻辑：
```C
while (1) {
        rt_event_recv(key_event, EVENT_FLAG_FIRST_CLICK | EVENT_FLAG_SECOND_CLICK,
                      RT_EVENT_FLAG_OR, RT_WAITING_FOREVER, &status);

        if (key_event->set == EVENT_FLAG_FIRST_CLICK) {
            rt_thread_mdelay(DISTURBANCE_TIME);

            if (KEY_PRESSED) {
                wait_flag = 1;
                key_ts_1 = rt_tick_get();
                while (KEY_PRESSED) {
                    rt_thread_mdelay(5);
                    if (rt_tick_get() - key_ts_1 >= LONG_PRESS_TIME) {
                        key_stat
