# Downloading Logs

This article explains how to transfer blackbox data from the board to an external reader over UART. The reader stores the received data on an SD card.

## Transfer Mode

1. Connect the reader to the designated UART port.
2. The firmware detects the connection and enters a dedicated "download" state. If the reader is already connected when the MCU powers up, the mission logic never starts and the transfer runs immediately.
3. The board reads log blocks from the external flash and sends them sequentially over UART.
4. Each block includes a sequence number and CRC.
5. The reader verifies the CRC and responds with `ACK` when the block is valid. If the CRC does not match or the block is lost, it sends `NACK` and the board resends that block.
6. After the final block is acknowledged the board exits transfer mode and resumes normal operation.

The download mechanism does not pause an already running mission. Instead, no mission code runs when a reader is detected at startup. Disconnect the reader and reboot once the transfer completes to start the scenario.

## Implementation Notes

- Block size is configured in `blackbox_config.h` and must equal `BBX_BUF_SIZE` (4096 bytes).
- Packets transmitted over UART are limited to `BBX_TX_SIZE` bytes to
  reduce RAM usage on the MCU. The transfer routine reads the flash
  sequentially and splits each block into these packets.
- All writes use the payload length reported in the block header so the
  total log size is unknown beforehand.
- The reader side only needs basic UART receive and SD write functionality.
- Reader presence is checked on the pin defined by
  `BBX_DET_PORT`/`BBX_DET_PIN` in `board_config.h`. If the level matches
  `BBX_DET_ACTIVE` at startup the firmware sends the log before running
  any scenario.
- Before any block data is transmitted the board sends a packet with
  sequence number `0xFFFF` containing the current session ID. The
  receiver can use this value to name the output file.
- Log data starts one block after address `0`, which stores the latest
  blackbox identifier. The firmware reads blocks sequentially until the
  following block header no longer matches the current session ID or its
  block number is not the expected next value. Each transferred block
  contains this identifier, its own sequence number and the payload
  length from the header. The length is verified against the maximum
  payload size before reading the block. Packets are acknowledged with
  `ACK`/`NACK` so corrupted chunks are resent automatically.

## Example Usage

Call the transfer routine after flushing the logger:

```c
blackbox_flush();
blackbox_transfer();
```

The host should acknowledge each block with byte `0x06` and send `0x15` to
request a resend when the CRC does not match.

## Implementing a Standalone Reader

A dedicated reader device can capture log data without a PC. Any MCU with
a UART and removable storage is sufficient. Configure the UART to the same
baud rate used by the board's debug port (115200 baud on the reference
hardware) and follow the sequence below:

1. Initialize the UART and mount the SD card or other storage medium.
2. Wait for the session ID packet (sequence number `0xFFFF`) and store the ID.
3. Wait for the board to transmit a block header.
4. Verify the header CRC and send `NACK` if it does not match.
5. Receive the payload in `BBX_TX_SIZE` chunks, acknowledging each with `ACK`.
6. Append the payload bytes to a log file.
7. Repeat for the next block until the board stops transmitting.

The reader does not know the final log size ahead of time. It keeps
tracking the sequence number and exits when it no longer increments.
Below is a simplified pseudo-code outline:

```c
open_uart(115200);
Packet id = read_packet();
if (id.seq != 0xFFFF || id.len != 1) return;
uint8_t session = id.data[0];
send_ack();
open_file_for_session(session);
uint16_t expected = 0;

while (true) {
    Header h = read_header();
    if (!crc_ok(h)) { send_nack(); continue; }
    if (h.block_num != expected)
        break; /* end of log */
    write_file(&h, sizeof(h));

    uint16_t received = 0;
    while (received < h.payload_len) {
        Packet p = read_packet();
        if (!crc_ok(p)) { send_nack(); continue; }
        write_file(p.data, p.len);
        received += p.len;
        send_ack();
    }
    expected++;
}
close_file();
```

This example omits hardware specifics but demonstrates the flow for a
simple external reader.

### Helper Functions

External readers typically implement a few small helpers for ACK/NACK
responses and CRC verification. The example below shows minimal
implementations that work with any UART driver.

```c
#define ACK  0x06
#define NACK 0x15

static void send_byte(uint8_t b)
{
    uart_write(bbx_uart, &b, 1); /* replace with your UART function */
}

void send_ack(void)  { send_byte(ACK); }
void send_nack(void) { send_byte(NACK); }

static uint8_t crc_table[256];

void crc_init(void)
{
    for (int i = 0; i < 256; ++i) {
        uint8_t r = i;
        for (int bit = 0; bit < 8; ++bit)
            r = (r & 0x80) ? (r << 1) ^ 0xD5 : (r << 1);
        crc_table[i] = r;
    }
}

static uint8_t crc_calc(const uint8_t *buf, int len)
{
    uint8_t crc = 0;
    for (int i = 0; i < len; ++i)
        crc = crc_table[buf[i] ^ crc];
    return crc;
}

bool crc_ok(const uint8_t *data, int len)
{
    uint8_t expected = data[len - 1];
    return crc_calc(data, len - 1) == expected;
}
```

Call `crc_init()` once during startup before verifying packets.

