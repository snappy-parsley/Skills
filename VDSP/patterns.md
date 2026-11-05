# Patterns for `visualdsp-rules`

Section numbers match the rules in `SKILL.md`. Snippets use DAQ5b's real primitives (`HALOS\types.h`, `messages.h`, `timekeeping.h`, `semaphore.h`, `OSAL\osal_error.h`). Names like `CAL_STATUS` and `Cal_*` stand in for your module.

## 1. Module status, OSAL and HALOS boundaries

```cpp
typedef enum {
    CAL_OK = 0,
    CAL_INVALID_ARGUMENT,
    CAL_NOT_FOUND,
    CAL_NO_MEMORY,
    CAL_TIMEOUT,
    CAL_BUSY,
    CAL_OS_ERROR
} CAL_STATUS;

CAL_STATUS Cal_Find_Channel(u16 id, CHANNEL_INFO *out)
{
    if (out == NULL || id >= NUM_CHANNELS)
        return CAL_INVALID_ARGUMENT;
    if (!channel_table[id].present)
        return CAL_NOT_FOUND;
    *out = channel_table[id];
    return CAL_OK;
}
```

OSAL returns `bool` and sets a last-error. Read it immediately and convert:

```cpp
static CAL_STATUS From_Osal(intptr_t err)
{
    switch (err) {
    case osalERROR_TIMEOUT:          return CAL_TIMEOUT;
    case osalERROR_OUT_OF_MEMORY:    return CAL_NO_MEMORY;
    case osalERROR_BUSY:             return CAL_BUSY;
    case osalERROR_INVALID_ARGUMENT:
    case osalERROR_BAD_POINTER:      return CAL_INVALID_ARGUMENT;
    default:                         return CAL_OS_ERROR;
    }
}

CAL_STATUS Cal_Wait_Ready(osalEvent_h ev, osalTime_ms_t timeout_ms)
{
    if (!osalWaitEvent(ev, timeout_ms))
        return From_Osal(osalGetLastError());
    return CAL_OK;
}
```

HALOS returns nonzero on success. Convert at the call site:

```cpp
CAL_STATUS Cal_Request_Start(void)
{
    if (!halos_Send_Message(cal_start_msg))   // 0 == failure
        return CAL_BUSY;
    return CAL_OK;
}
```

`new` returns `NULL` on exhaustion because exceptions are off:

```cpp
cal_queue = new HALOS_MESSAGE_QUEUE(CAL_QUEUE_LENGTH, cal_task_id);
if (cal_queue == NULL) {
    Print_to_Sys_Log("Cal: queue alloc failed");
    return CAL_NO_MEMORY;
}
```

## 2. Closed domains and message dispatch

```cpp
typedef enum { GAIN_X1, GAIN_X4, GAIN_X16 } GAIN_SEL;

CAL_STATUS Gain_To_Register(GAIN_SEL g, u8 *out)
{
    switch (g) {
    case GAIN_X1:  *out = 0x00; return CAL_OK;
    case GAIN_X4:  *out = 0x01; return CAL_OK;
    case GAIN_X16: *out = 0x03; return CAL_OK;
    default:
        Print_to_Sys_Log("Gain: bad selector %u", (u32)g);
        return CAL_INVALID_ARGUMENT;
    }
}
```

Message ids are runtime values, so dispatch is an `if / else if` chain. Every branch is `else if`, and the final `else` reports the unknown id:

```cpp
while (cal_queue->Get(hm)) {
    if (hm.msg_id == cal_start_msg.msg_id)
        Cal_Handle_Start();
    else if (hm.msg_id == cal_timer->Msg_Id())
        Cal_Handle_Timeout();
    else if (hm.msg_id == Trigger_Msg_Id())
        Cal_Handle_Trigger(*(const TRIGGER_EVENT *)hm.msg);
    else
        Print_to_Sys_Log("Cal: unknown msg %u", (u32)hm.msg_id);
}
```

## 3. Ownership

Document the `msg` contract next to the accessor in the header:

```cpp
// msg: CAL_RESULT* owned by cal module (static pool); valid until the next
//      Cal_Result_Msg for the same slot. Receiver copies, does not delete.
u16 Cal_Result_Msg_Id(void);

// msg: (void*)(u32)channel. Scalar, no ownership.
u16 Cal_Channel_Msg_Id(void);

// msg: DATA_PACKET* from new. Receiver takes ownership and deletes.
//      Registered MESSAGE_BUFFERED_EVERY_TIME, single receiver queue.
u16 Cal_Packet_Msg_Id(void);
```

Wrong, because the pointer dies when the function returns:

```cpp
void Send_Event(u64 t_ns)
{
    TRIGGER_EVENT ev;            // stack
    ev.trigger_time_nsec = t_ns;
    hm.msg = (void *)&ev;        // dangling once delivered
    halos_Send_Message(hm);
}
```

Use a `static` or pool slot that outlives delivery, or pass a scalar.

Partial setup undoes in reverse order. `delete NULL` is a no-op:

```cpp
CAL_STATUS Cal_Init(void)
{
    cal_queue = new HALOS_MESSAGE_QUEUE(CAL_QUEUE_LENGTH, cal_task_id);
    cal_timer = new TIMER_CLASS(CAL_TIMEOUT_MS, TIMER_SINGLE_SHOT, TIMER_STOPPED);
    if (cal_queue == NULL || cal_timer == NULL) {
        delete cal_timer;  cal_timer = NULL;
        delete cal_queue;  cal_queue = NULL;
        return CAL_NO_MEMORY;
    }
    return CAL_OK;
}
```

Borrowed buffer: the callee neither deletes nor stores it.

```cpp
CAL_STATUS Cal_Parse(const u8 *data, u32 len);
```

## 4. Task skeleton

```cpp
static u16 cal_task_id;
static HALOS_MESSAGE_QUEUE *cal_queue = NULL;
static HALOS_MESSAGE cal_start_msg;
static TIMER_CLASS *cal_timer = NULL;

static void Cal_Task(void *);

void Cal_Task_Init(void)
{
    cal_task_id = CreateTask(Cal_Task, NULL, CAL_TASK_STACK_SIZE,
                             CAL_TASK_PRIORITY, TASK_SLEEP, (u8 *)"Cal_Task");
    cal_queue = new HALOS_MESSAGE_QUEUE(CAL_QUEUE_LENGTH, cal_task_id);
    cal_timer = new TIMER_CLASS(CAL_TIMEOUT_MS, TIMER_SINGLE_SHOT, TIMER_STOPPED);

    cal_start_msg.msg_id = halos_Register_Message(CAL_QUEUE_LENGTH);
    cal_start_msg.msg = NULL;

    halos_Register_Message_Queue(cal_start_msg.msg_id, MESSAGE_BUFFERED_ONE_TIME, cal_queue);
    halos_Register_Message_Queue(cal_timer->Msg_Id(), MESSAGE_BUFFERED_ONE_TIME, cal_queue);
}

static void Cal_Task(void *)
{
    HALOS_MESSAGE hm;

    for (;;) {
        while (cal_queue->Get(hm)) {
            /* dispatch, see section 2 */
        }
        TaskSleep();
    }
}
```

Add stack size and priority to `Config\main_defs.h` / `Config\task_priorities.h`, and call `Cal_Task_Init()` in `main.cpp` after its dependencies and before `Enable_Multitasking()`.

Shared state lives in one module with one lock:

```cpp
static SEMAPHORE meter_lock;     // guards meter_count, meter_samples[]
static u16 meter_count = 0;
static s16 meter_samples[METER_LEN];

void Meter_Add(s16 sample)
{
    meter_lock.EnterCriticalSection();
    if (meter_count < METER_LEN)
        meter_samples[meter_count++] = sample;
    meter_lock.LeaveCriticalSection();
}
```

## 5. Waiting without blocking the handler

Wrong, because every sender to this queue stalls for up to 2 s:

```cpp
void Cal_Handle_Start(void)
{
    Start_Hardware_Cal();
    while (!Hardware_Cal_Done())
        Sleep(10);
    Report_Cal_Result();
}
```

Right: arm a single-shot timer and handle completion as a message.

```cpp
static u8 cal_polls;

void Cal_Handle_Start(void)
{
    Start_Hardware_Cal();
    cal_polls = 0;
    cal_timer->Period(CAL_POLL_MS);
    cal_timer->Restart();
}

void Cal_Handle_Timeout(void)
{
    if (Hardware_Cal_Done()) {
        Report_Cal_Result();
    } else if (++cal_polls >= CAL_MAX_POLLS) {
        Report_Cal_Status(CAL_TIMEOUT);
    } else {
        cal_timer->Restart();
    }
}
```

## 6. Interrupt handlers

```cpp
static volatile u16 ppi_wr_idx = 0;        // written in ISR, read by task
static volatile u32 ppi_overruns = 0;

static void PPI_Int_Handler(void *)
{
    if (*pDMA0_IRQ_STATUS & DMA_DONE) {
        *pDMA0_IRQ_STATUS = DMA_DONE;      // acknowledge first
        if (++ppi_wr_idx >= NUM_ADC_BUFFERS)
            ppi_wr_idx = 0;
    }
    /* no new/delete, Sleep, SEMAPHORE, file/network I/O, or Print_to_Sys_Log */
}

void Read_Ppi_Stats(u16 *idx, u32 *overruns)
{
    u32 imask = cli();                     // consistent snapshot of both fields
    *idx = ppi_wr_idx;
    *overruns = ppi_overruns;
    sti(imask);
}
```

Callbacks registered with a driver that calls them from its ISR (as `FPGA_PPI.cpp` does) obey the same restrictions.

## 7. Variants

```cpp
#if defined(DAQ12)
static const u8 NUM_CHANNELS = 12;
#else
static const u8 NUM_CHANNELS = 24;
#endif

#ifdef BIRD_DOG
    Bird_Dog_Specific_Init();
#endif
```

Add a function that takes a struct rather than a base class with one virtual implementation.

## 8. Hot-path layout and named constants

```cpp
typedef struct {
    s32 last;
    u16 flags;
} CHANNEL_SLOT;

static CHANNEL_SLOT channel_slots[NUM_CHANNELS];

s32 Channel_Last(u8 index)          // an index stays valid if the table is rebuilt
{
    return channel_slots[index].last;
}
```

```cpp
static const u8 CMD_NOP         = 0x00;
static const u8 CMD_START       = 0x01;
static const u8 CMD_STOP        = 0x02;
static const u8 CMD_READ_STATUS = 0x03;
```

Before adding an abstraction, ask whether a function and a struct are enough, and whether a second implementation exists. If not, keep the direct call.
