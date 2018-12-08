# Embedded Challenge - 1

## scheme

Amtega MCU w/ external SPI NOR flash and a ublox module (UART) and a SPI Digital Accelerometer.

![scheme](01/scheme.png)

It looks like scheme is a bit incorrect - UART to the MCU is connected to two GNDs?

Just from a brief look the system is quite understandable. I assume that NMEA messages are sent via UART to the MCU, which parses those and write them via SPI to flash.I also assume there may be some thread / interrupt that involves other writes to the flash.

So the areas in the code that might be interesting are:
  1. UART
  2. SPI
  3. NMEA message parsing

## Parsing the Memory Dump

Looking at the main loop - the `parse` function reads NMEA data. It supports both `GPGGA` and `GPRMC`.
After reading some specifications the interesting fields for each type are:
  * `GPGGA`: lat, long and time
  * `GPRMC`: time

*You can read more about NMEA message format [here](http://aprs.gids.nl/nmea/)*

`parse` checks if the given NMEA message in GPGGA or GPRMC - returns `1` for `GPGGA` and `2` for `GPRMC`. It also copies the message components (i.e. message.split(',')) to the `out_buffers`.

Later on, `format1` and `format2` functions are being called to parse the NMEA data. 
`format1` is for GPRMC, which saves two ints that represents date and hour:
```c
// GPRMC
static uint32_t format1(uint8_t **in_buffer, uint8_t *out_buffer)
{
    *(int *)out_buffer = atoi(in_buffer[0]);
    out_buffer += sizeof(float);
    *(int *)out_buffer = atoi(in_buffer[8]);
    return 2 * sizeof(float);
}

```
`format2` is done for `GPGGA` - which parses the long and lat to degress and then saves them as two floats. 
```c
// GPGGA
static uint32_t format2(uint8_t **in_buffers, uint8_t *out_buffer)
{
    // i = 0: Latitude: {N,S}
    // i = 2: Longtitude {E, W}
    char val1_str[10] = { 0 };
    for (uint8_t i = 0; i < 3; i += 2)
    {
        memset(val1_str, 0, sizeof(val1_str));
        uint8_t *pos = strchr(in_buffers[i], '.');
        memcpy(val1_str, in_buffers[i], pos - in_buffers[i] - 2);
        // lat and long
        uint16_t val1 = atoi(val1_str);
        float val2 = atof(pos - 2); // take the number / 100
        float res =  val1 + (val2 / 60); // take the number % 100 + float
        if (*in_buffers[i + 1] == 'W' || *in_buffers[i + 1] == 'S')
            res *= -1;
        *(float *)out_buffer = res;
        out_buffer += sizeof(float);
    }
    return 2 * sizeof(float);
}
```
It took me a WHILE to understand that `format2` just transforms and longtitude and latitude and does not encode them.

here is a screenshot of a debugging session in order to understand `format2`:

![format2 debug](01/format2_debug.png)

The `format_save` function appears to format the data before it is written to the flash:
```c
static uint32_t format_save(uint8_t *in_buffer, uint8_t a, uint32_t length, uint8_t *out_buffer)
{
    *out_buffer = a;
    *(uint32_t *)&out_buffer[1] = length;
    memcpy(&out_buffer[5], in_buffer, length);
    return length + 5;
}
```

The result is saved to `out_buffer`. The first byte consists of the message ID, the next 4 bytes represent length and `length` bytes afterwards represents the message payload itself.


`format_save` is called from the following places in the code:

  * The `main` routine:

    ```c
    if(formatted_length > 0)
    {
        save_length = format_save(temp_buffer, (a == TRUE) ? 0 : 1, formatted_length, save_buffer);
        save_to_flash(save_buffer, save_length);
    }
    ```
    a message with the payload `temp_buffer` - which is the result of the `parse` function and an id of either `TRUE` (1) or `FALSE` (0) is being saved to the flash - depending on the type of the NMEA message written.
  * an Interrupt Service Routine (ISR):

    ```c
    ISR(INT2_vect)
    {
        static uint8_t interrupt_buffer[MAX_SAVE_BUFFER_SIZE];
        uint8_t *temp_buffer;
        uint8_t a;
        uint32_t length;
        if (PIND & 0x01)
        {
            is_triggered = TRUE;
            a = 2;
        }
        else
        {
            is_triggered = FALSE;
            a = 3;
        }
    
        length = format_save(temp_buffer, a, 0, interrupt_buffer);
        save_to_flash(interrupt_buffer, length);
    }
    ```
    depending on the triggered pin (I assume), `is_triggered` changes value and an empty message with id `2` or id `3` is being saved to the external memory. 

With that information - we can write some basic parser! Here is a snippet of a python parser I wrote:
```python
FlashMessageHeader = struct.Struct("<"+"BI")

class FlashMessage(object):
    class GPRMC(object):
        def __init__(self, data):
            GPData = struct.Struct("<"+"II")
            self.time, self.date = GPData.unpack(data[:GPData.size])
            data = data[GPData.size:]

        def __str__(self):
            return("GPRMC:time:{time}\t,data:{date}\t").format(time=self.time, date=self.date)

    class GPGGA(object):
        def __init__(self, data):
            GPData = struct.Struct(ENDIANESS+"ff")
            self.lat, self.long = GPData.unpack(data[:GPData.size])
            data = data[:GPData.size:]

        def __str__(self):
            return("GPGGA: lat:{lat:.5f}, \tlong:{long:.5f}\t").format(lat=self.lat, long=self.long)
```
and here is a screenshot of the parsed external memory:
![parsed memory](01/initial_parser.png)

*Taking a random lat/long line to google maps gives a coordinate in Israel!*

Taken from that: We can get some **timestamps** based upon the `GPRMC` messages!
We need to get the `GPGGA` message that was recorded at the time the SMS message was sent.

## Timestamps

Well, how can we do that? If we continute reading the code - `GPGGA` messages are only saved if `should_save == TRUE`.
`should_save` is being updated by the following ISR:
```c
ISR(TIMER1_COMPA_vect)
{
    if (++counter == counter_max_val)
    {
        should_save = TRUE;
        counter = 0;
        counter_max_val = (is_triggered == TRUE) ? 15 : 150;
    }
}
```

From the looks of it - its a timer ISR, which, when reaches the counter, enables the logging of a `GPGGA` message.
If we figure out the frequency of that ISR, we can then get some sort of timestamp applied to each `GRGGA` message. With an absolute timestamp that we can get from a `GPRMC` message -an absolute timestamp can be calculated for each `GPGGA` message! 

### Timer ISRs 101
Each timer has a special "comparator" register. On each "tick" the comparator register is incremented. When the comparator hits a certain presetted value, the ISR is being executed.

Since the MCU may have a different clocks - which can be based on the same internal frequency (the CPUs frequency) - each clock has a component called a prescalar, which is as a divisor to that clock frequency.

In order to figure out the frequency, we need three components:

  * The clock frequency
  * The comparator register value
  * The prescalar value

And the timer ISR frequency itself would be:
` Frequency = (CLOCK_FREQUENCY / PRESCALAR) / (COMPARATOR + 1)`



Now that we got that out of the way:

  * The clock frequency is at 16Mhz and is documented in the code:
    ```c
    configure_sysclk();	/* Assume system clock is 16 MHz */
    ```

  * Both the comparator and the prescalar are set in `configure1`:
    ```c
    static void configure1(void)
    {
        cli();
        should_save = FALSE;
        counter = 0;
        TCCR1A = 0;
        TCCR1B = 0;
        TCNT1 = 0;
        OCR1A = 62499;
        TCCR1B |= (1 << WGM12);
        TCCR1B |= (1 << CS12) | (1 << CS10);
    }
    ```

After reading the datasheet: `OCR1A` is the comparator, and TCCR1B can control the prescalar:

![prescalar from datasheet](01/prescalar.png)

Meaning that the prescalar = 1024

Hence the timer ISR frequency:

![ipython ISR freq](01/isr_py.png)

That means that every increase of the counter variable == 4 seconds!
It is also seen that:
  * Each `GPGGA` message resets `counter`
  * Messages with id 2 sets `counter_max_val` to 15
  * Messages with id 3 sets `counter_max_val` to 150
  * Each `GPRMC` message resets the accumalated time - because it happend after a reset, so `counter_max_val` is set to 75

with that information - I added a `TICK` field to the parser:
Upon a new message:
```python
if self._type == 0:
    self.gpmsg = FlashMessage.GPRMC(self._payload)
    FlashMessage.TICK = 75
    FlashMessage.NEXT_TICK = 150
elif self._type == 1:
    self.gpmsg = FlashMessage.GPGGA(self._payload)
    FlashMessage.TICK += FlashMessage.NEXT_TICK
elif self._type == 2:
    self.gpmsg = None
    FlashMessage.NEXT_TICK = 15
elif self._type == 3:
    FlashMessage.NEXT_TICK = 150
    self.gpmsg = None
```

Now the parsed messaged looks like this:

![parsed messages](01/parsed_with_tick.png)

We also notice that a reset occured, and the MCU latter started at `28/10/18:16:17`:

![GPRMC after reset](01/gprmc_after_reset.png)

We know that the message was recived at `30/10/18:01:20`, we can use the power of python in order to get the relevant tick!

![tick calculation](01/tick_calc.png)

And the answer is the following:

![answer](01/answer.png)

