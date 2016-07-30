# openSPOT HTTP API

The HTTP API uses JSON queries and replies.

All queries except gettok.cgi must include a valid token and digest. To acquire these, your application must complete the login process. These two keys are not represented in the JSON structure descriptions below.

## Login process

- Request a token using *gettok.cgi*.
- Concatenate the token and the password string.
- Hash this using SHA256 to get the digest.
- Call login.cgi with the token and the digest in the query.

Example login process with example JSON queries:

- POST **gettok.cgi** an empty query. Response:
	```json
	{
  	"token": "1f9a8b7c"
	}
	```
- Our password is *"passw0rd"*. So we concatenate the token and the password, and hash it:
	```bash
	sha256("1f9a8b7cpassw0rd")
	```
- This gives us the digest *"2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"*
- Now we call **login.cgi** with this JSON query:
	```json
	{
  	"token": "1f9a8b7c",
  	"digest": "2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"
	}
	```
	The reply will be:
	```json
	{
  	"success": 1,
  	"hostname": "openspot"
	}
	```
- Now we are logged in and can call all API interfaces with the *token* and *digest*.

## API interfaces

### gettok.cgi

Doesn't take any parameters from a query. Returns the session token, which is a hexadecimal uint32_t (8 ASCII characters).

Response:
```json
{
  "token": "1f9a8b7c"
}
```

### checkauth.cgi

Checks the validity of the given token and digest. *success* is 1 if they are valid **and** the user has logged in previously. Also returns openSPOT's hostname and current IP address.

Response:
```json
{
  "success": 1,
  "hostname": "openspot",
  "ip_address": "192.168.3.99"
}
```

### login.cgi

Logs in the user if the given token and digest is valid. *success* is 1 if they are valid. Also returns openSPOT's hostname.

Response:
```json
{
  "success": 1,
  "hostname": "openspot"
}
```

### logout.cgi

Logs out the user if the given token and digest is valid **and** the user has logged in previously. *success* is 1 if logout was successful.

Response:
```json
{
  "success": 1
}
```

### reboot.cgi

Checks the validity of the given token and digest, and triggers a reboot. *success* is 1 if token and digest are valid **and** the user has logged in previously.

If *reset_config* is set to 1 in the query, it resets the default configuration after rebooting.

Query (optional):
```json
{
  "reset_config": 1
}
```
Response:
```json
{
  "success": 1
}
```

### status.cgi

Returns openSPOT's current status.

**status** can be:
  - 0: Standby
  - 1: In call
  - 2: Connector not set
  - 3: Connector connecting
  - 4: Modem initializing
  - 5: Modem disconnected
  - 6: Modem HW/SW version mismatch
  - 7: Modem firmware upgrade in progress

**rssi_tc0_values_dbm** and **rssi_tc1_values_dbm** contain RSSI values since the last call of *status.cgi*. If the current modem mode is non-TDMA, then ignore *rssi_tc1_values_dbm*.

**dejitter_buf_tc0_pkts** and **dejitter_buf_tc1_pkts** contain dejitter buffer packet count values since the last call of *status.cgi*. If the current modem mode is non-TDMA, then ignore *dejitter_buf_tc1_pkts*.

The packet and byte UDP traffic counters are monotonically increasing 32 bit values.

Response:
```json
{
  "status": 0,
  "rssi_tc0_values_dbm": [-60,-62,-65],
  "rssi_tc1_values_dbm": [-60,-62,-65],
  "dejitter_buf_tc0_pkts": [0, 1, 2],
  "dejitter_buf_tc1_pkts": [0, 1, 2],
  "ber_tc0_values": [0, 1, 2],
  "ber_tc1_values": [0, 1, 2],
  "invalid_seqnums": 5,
  "rx_pkts": 32,
  "rx_bytes": 14421,
  "tx_pkts": 32,
  "tx_bytes": 14421
}
```

### status-dmrsms.cgi

Returns current status of DMR SMS sending, if the modem is in DMR mode. Otherwise it'll return with 400 Bad Request.
Also it handles SMS sending.

Call types: 0 - private, 1 - group.
Format IDs: 0 - ETSI, 1 - UDP. See user manual for more info.
If a new message is received, *rx_msg_valid* is 1.
Setting a new *send_srcid* in the query overwrites the *default_srcid*.

Messages are in hexadecimal UTF16BE format. Example: "BEER" = "0042004500450052"
Max. message length which can be sent is currently 75 UTF16BE characters (150 hex char pairs).

Query (optional):
```json
{
  "send_dstid": 2161005,
  "send_calltype": 0,
  "send_srcid": 9998,
  "send_format": 0,
  "send_msg": "0042004500450052"
}
```

Response:
```json
{
  "default_srcid": 9998,
  "send_ongoing": 0,
  "send_success": 1,
  "send_fail": 0,
  "rx_msg_valid": 0
  "rx_msg_srcid": 1234
  "rx_msg_calltype": 0,
  "rx_msg_format": 0,
  "rx_msg": "0042004500450052"
}
```

### status-srfipconnserver.cgi

Returns the SharkRF IP Connector Server's current status, if it's the active connector. Otherwise it'll return with 400 Bad Request.

**client_connected** is 1 if a client is connected.

Response:
```json
{
  "client_connected": 0,
  "client_id": 1234,
  "client_callsign": ""
}
```

### connector.cgi

If you want to change openSPOT's active connector, you can POST a query to this CGI. *changed* is 1 if the active connector is changed.

Valid connector IDs:

- 0: No connector set.
- 1: DMRplus
- 2: Homebrew
- 3: TS repeat
- 4: DCS
- 5: FCS
- 6: SharkRF IP Connector Client
- 7: SharkRF IP Connector Server

Query (optional):
```json
{
  "new_connector": 0
}
```
Response:
```json
{
  "changed": 0,
  "active_connector": 0
}
```

### dmrplussettings.cgi

If you want to change the DMRplus connector settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_rx_freq": 436000000,
  "new_tx_freq": 436000000,
  "new_server_host": "",
  "new_port": 8880,
  "new_dmr_id": "",
  "new_reflector_id": 0,
  "new_keepalive_interval_sec": 1,
  "new_rx_timeout_sec": 10
}
```
Response:
```json
{
  "changed": 1,
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 8880,
  "dmr_id": "",
  "reflector_id": 0,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10
}
```

### homebrewsettings.cgi

If you want to change the Homebrew connector settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Valid *c4fm_dstcalltype* values: 0 - group call, 1 - private call.

Query (optional):
```json
{
  "new_rx_freq": 436000000,
  "new_tx_freq": 436000000,
  "new_server_host": "",
  "new_port": 62030,
  "new_callsign": "",
  "new_password": "",
  "new_repeater_id": 901234,
  "new_autocon_id": 4771,
  "new_autocon_tdma_channel": 0,
  "new_autocon_interval_sec": 500,
  "new_dmo_tdma_channel": 0,
  "new_c4fm_dstid": 9,
  "new_c4fm_dstcalltype": 0,
  "new_keepalive_interval_sec": 5,
  "new_rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "changed": 1,
  "server_host": "",
  "port": 62030,
  "callsign": "",
  "password": "",
  "repeater_id": 901234,
  "autocon_id": 4771,
  "autocon_tdma_channel": 0,
  "autocon_interval_sec": 500,
  "dmo_tdma_channel": 0,
  "c4fm_dstid": 9,
  "c4fm_dstcalltype": 0,
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30
}
```

### dcssettings.cgi

If you want to change the DCS connector settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_rx_freq": 436000000,
  "new_tx_freq": 436000000,
  "new_server_host": "",
  "new_port": 12345,
  "new_ccs_port": 12345,
  "new_callsign": "",
  "new_local_module": "A",
  "new_reflector": "DCS025",
  "new_remote_module": "Z",
  "new_rx_timeout_sec": 30
}
```
Response:
```json
{
  "changed": 1,
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "A",
  "reflector": "DCS025",
  "remote_module": "Z",
  "rx_timeout_sec": 1
}
```

### fcssettings.cgi

If you want to change the FCS connector settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_server_host": "fcs001.xreflector.net",
  "new_rx_freq": 436000000,
  "new_tx_freq": 436000000,
  "new_port": 12345,
  "new_callsign": "",
  "new_ccs7_id": 2161005,
  "new_reflector": "FCS001",
  "new_room_number": 25,
  "new_keepalive_interval_sec: 1,
  "new_rx_timeout_sec": 30
}
```
Response:
```json
{
  "changed": 1,
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "fcs001.xreflector.net",
  "port": 12345,
  "callsign": "",
  "ccs7_id": 2161005,
  "reflector": "FCS001",
  "room_number": 25,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10
}
```

### srfipconnclientsettings.cgi

If you want to change the SharkRF IP Connector Client settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_rx_freq": 436000000,
  "new_tx_freq": 436000000,
  "new_server_host": "192.168.3.120",
  "new_port": 65100,
  "new_id": 2161005,
  "new_password": "abcdefgh",
  "new_callsign": "",
  "new_keepalive_interval_sec: 5,
  "new_rx_timeout_sec": 30
}
```
Response:
```json
{
  "changed": 1,
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "192.168.3.120",
  "port": 65100,
  "id": 2161005,
  "password": "abcdefgh",
  "callsign": "",
  "keepalive_interval_sec: 5,
  "rx_timeout_sec": 30
}
```

### srfipconnserversettings.cgi

If you want to change the SharkRF IP Connector Server settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_rx_freq": 436000000,
  "new_tx_freq": 436000000,
  "new_port": 65100,
  "new_password": "abcdefgh",
  "new_rx_timeout_sec": 30
}
```
Response:
```json
{
  "changed": 1,
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "port": 65100,
  "password": "abcdefgh",
  "rx_timeout_sec": 30
}
```

### info.cgi

Allows you to query general info about the openSPOT box. *blver* is the bootloader version. *uid* is the device unique ID in hexadecimal.

Response:
```json
{
  "hwver": "1.0",
  "swver": "0001",
  "blver": "0001",
  "mac": "FE:28:00:00:00:FA",
  "uid": "abcdef"
}
```

### passwordsettings.cgi

Allows you to change openSPOT's password. *changed* is 1 if the password has been changed.

Query:
```json
{
  "new_password": "openspot",
}
```
Response:
```json
{
  "changed": 1
}
```

### netsettings.cgi

If you want to change the network settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

**ip_config_mode** can be:
  - 0: DHCP
  - 1: DHCP with auto IP
  - 2: Auto IP
  - 3: Static IP

Query (optional):
```json
{
  "new_ip_config_mode": 0,
  "new_hostname": "openspot",
  "new_static_ip": "192.168.1.99",
  "new_static_mask": "255.255.255.0",
  "new_static_gw": "192.168.1.1",
  "new_static_dns1": "8.8.8.8",
  "new_static_dns2": "8.8.4.4",
  "new_dejitter_queue_msec": 130
}
```
Response:
```json
{
  "changed": 1,
  "ip_config_mode": 0,
  "hostname": "openspot",
  "static_ip": "192.168.1.99",
  "static_mask": "255.255.255.0",
  "static_gw": "192.168.1.1",
  "static_dns1": "8.8.8.8",
  "static_dns2": "8.8.4.4",
  "dejitter_queue_msec": 130
}
```

### locationsettings.cgi

If you want to change location settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_latitude": "0.0",
  "new_longitude": "0.0",
  "new_height_agl": 0,
  "new_name": ""
}
```
Response:
```json
{
  "changed": 1,
  "latitude": "0.0",
  "longitude": "0.0",
  "height_agl": 0,
  "name": ""
}
```

### dmrsettings.cgi

If you want to change general DMR settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_cc": 1,
  "new_echo_id": 9999,
  "new_default_c4fm_id": 0,
  "new_transmit_idle_in_idle_tx_tdma_channel": 1
}
```
Response:
```json
{
  "changed": 1,
  "cc": 1,
  "echo_id": 9999,
  "default_c4fm_id": 0,
  "transmit_idle_in_idle_tx_tdma_channel": 1
}
```

### dstarsettings.cgi

If you want to change general D-STAR settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings.

Query (optional):
```json
{
  "new_echo_callsign": "       E"
}
```
Response:
```json
{
  "changed": 1,
  "echo_callsign": "       E"
}
```

### modemfreq.cgi

If you want to change the current RX, TX frequency or TX power without reinitializing the modem, you can POST a query to this CGI. *changed* is 1 if the frequency got changed. Returns currently active settings.

DMR demodulation mode values: 0 - A, 1 - B, 2 - C etc.

Query (optional):
```json
{
  "new_rx_frequency": 433450000,
  "new_dmr_demodmode": 0,
  "new_tx_frequency": 433450000,
  "new_tx_power_percent": 100
}
```
Response:
```json
{
  "changed": 1,
  "rx_frequency": 433450000,
  "dmr_demodmode": 0,
  "tx_frequency": 433450000,
  "tx_power_percent": 100
}
```

### modemmode.cgi

If you want to change the current modem mode, you can POST a query to this CGI. *changed* is 1 if the mode got changed. Returns currently active settings. *modem_init_delay_ms* is the time needed for the modem to calibrate and initialize.

- **mode** can be:
  - 0: Idle
  - 1: Raw
  - 2: DMR
  - 3: D-STAR
  - 4: C4FM

- **submode** can be:
  - 0: No submode set
  - 1: DMR Hotspot
  - 2: DMR MS
  - 3: DMR BS

Query (optional):
```json
{
  "new_mode": 0,
  "new_submode": 0,
}
```
Response:
```json
{
  "changed": 1,
  "modem_init_delay_ms": 3500,
  "mode": 0,
  "submode": 0,
}
```

### modemmodulation.cgi

If you want to change the current modulation mode, you can POST a query to this CGI. *changed* is 1 if the modulation got changed. Returns currently active settings. *modem_init_delay_ms* is the time needed for the modem to calibrate and initialize.

- **modulation_mode** can be:
  - 0: 2FSK
  - 1: 2FSK Raised Cosine
  - 2: 4FSK
  - 3: 4FSK Raised Cosine

Query (optional):
```json
{
  "new_modulation_mode": 0,
  "new_bitrate": 9600,
  "new_inner_deviation_hz": 648
}
```
Response:
```json
{
  "changed": 1,
  "modem_init_delay_ms": 3500,
  "modulation_mode": 0,
  "bitrate": 9600,
  "inner_deviation_hz": 648
}
```

### modempacket.cgi

If you want to change the current packet settings, you can POST a query to this CGI. *changed* is 1 if packet settings got changed. Returns currently active settings. *modem_init_delay_ms* is the time needed for the modem to calibrate and initialize.

Query (optional):
```json
{
  "new_packet_size_in_bits": 288,
  "new_sync_word_length_in_bits": 24,
  "new_sync_word_pos_in_packet_in_bits": 0,
  "new_sync_word_count": 1,
  "new_sync_word1": "4f5de8",
  "new_sync_word2": "445566"
}
```
Response:
```json
{
  "changed": 1,
  "modem_init_delay_ms": 3500,
  "packet_size_in_bits": 288,
  "sync_word_length_in_bits": 24,
  "sync_word_pos_in_packet_in_bits": 0,
  "sync_word_count": 1,
  "sync_word1": "4f5de8",
  "sync_word2": "445566"
}
```

### modempacket.cgi

If you want to change the current TDMA settings, you can POST a query to this CGI. *changed* is 1 if packet settings got changed. Returns currently active settings. *modem_init_delay_ms* is the time needed for the modem to calibrate and initialize.

Query (optional):
```json
{
  "new_tdma_enabled": 1,
  "new_tdma_pit_calibration_wait_for_packets_num": 2,
  "new_tdma_pit_calibration_compensation_multiplier": 1.2,
  "new_tdma_needed_sync_frames_for_tdma_channel_be_valid": 1
}
```
Response:
```json
{
  "changed": 1,
  "modem_init_delay_ms": 3500,
  "tdma_enabled": 1,
  "tdma_pit_calibration_wait_for_packets_num": 2,
  "tdma_pit_calibration_compensation_multiplier": 1.2,
  "tdma_needed_sync_frames_for_tdma_channel_be_valid": 1
}
```

### modemcal.cgi

If you want to change modem calibration settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings. *modem_init_delay_ms* is the time needed for the modem to calibrate and initialize.

Query (optional):
```json
{
  "new_auto_calibration": 1,
  "new_recalibrate_temp_diff_in_celsius": 10,
  "new_recalibrate_rf_ic_temp_diff_in_celsius": 10,
  "new_temp_read_interval_in_sec": 10,
  "new_temp_read_delay_in_sec_after_sync_lost": 10,
  "new_quick_calibrate_delay_in_sec_after_sync_lost": 10
}
```
Response:
```json
{
  "changed": 1,
  "modem_init_delay_ms": 3500,
  "auto_calibration": 1,
  "recalibrate_temp_diff_in_celsius": 10,
  "recalibrate_rf_ic_temp_diff_in_celsius": 10,
  "temp_read_interval_in_sec": 10,
  "temp_read_delay_in_sec_after_sync_lost": 10,
  "quick_calibrate_delay_in_sec_after_sync_lost": 10
}
```

### modemother.cgi

If you want to change other modem settings, you can POST a query to this CGI. *changed* is 1 if at least one setting got changed. Returns currently active settings. *modem_init_delay_ms* is the time needed for the modem to calibrate and initialize.

*agc_auto* and *external_vco* fields are booleans.

Query (optional):
```json
{
  "new_rssi_avg_sample_count": 5,
  "new_high_gain_low_linearity": 0,
  "new_agc_auto": 0,
  "new_agc_low_threshold_dbm": -50,
  "new_agc_high_threshold_dbm": -80,
  "new_external_vco": 0,
  "new_call_hang_time_ms": 3000
}
```
Response:
```json
{
  "changed": 1,
  "modem_init_delay_ms": 3500,
  "rssi_avg_sample_count": 5,
  "high_gain_low_linearity": 0,
  "agc_auto": 0,
  "agc_low_threshold_dbm": -50,
  "agc_high_threshold_dbm": -80,
  "external_vco": 0,
  "call_hang_time_ms": 3000
}
```

