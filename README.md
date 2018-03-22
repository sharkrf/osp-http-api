# openSPOT HTTP API

Note that this repository is always updated according to the beta openSPOT firmware releases.

The HTTP API uses JSON queries and replies.

All queries except gettok.cgi, ip.cgi and checkauth.cgi must include a valid JWT
([JSON Web Token](https://jwt.io)).
The JWT should be included in the HTTP header as Authorization: Bearer.

To acquire a JWT, your application must complete the login process. The JWT stays valid for
3600 seconds after the last valid query. If the supplied JWT is not valid, openSPOT responds
with 403 Forbidden header.

## Login process

- Request a token using *gettok.cgi*.
- Concatenate the token and the password string.
- Hash this using SHA256 to get the digest.
- Call login.cgi with the token and the digest in the query.

Example login process with example JSON queries:

- GET **gettok.cgi** an empty query. Response:
	```json
	{
	  "token": "1f9a8b7c"
	}
	```

- Our password is *"passw0rd"*. So we concatenate the token and the password, and hash it:
	```bash
	sha256("1f9a8b7cpassw0rd")
	```
	This gives us the digest *"2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"*

- Now we POST **login.cgi** this JSON:
	```json
	{
	  "token": "1f9a8b7c",
	  "digest": "2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"
	}
	```

	The reply will be:

	```json
	{
	  "hostname": "openspot",
	  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJkMjljODQwZSJ9.r3Oom8qVEAd1ceMMWibrMNsgu0DPgz-IG13MAzB-o5s"
	}
	```
	If the password is not matching, openSPOT will respond with a 401 Unauthorized header.

- Now we are logged in and can call all API interfaces with the acquired JWT.

## API interfaces

### ip.cgi

Returns the IP address of openSPOT.

Response:
```json
{
  "ip": "192.168.3.106"
}
```

### gettok.cgi

Doesn't take any parameters from a query. Returns the session token, which is
a hexadecimal uint32_t (8 ASCII characters).

Response:
```json
{
  "token": "1f9a8b7c"
}
```

### checkauth.cgi

Checks the validity of the supplied JWT. *success* is 1 if it's valid **and**
the user has logged in previously. Also returns openSPOT's hostname and current
IP address. *nopass* is 1 if openSPOT has no password set.

Response:
```json
{
  "success": 1,
  "nopass": 0,
  "hostname": "openspot",
  "ip_address": "192.168.3.99"
}
```

### login.cgi

Logs in the user if the given token and digest is valid. Returns openSPOT's hostname
and the JWT.

Response:
```json
{
  "hostname": "openspot",
  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJkMjljODQwZSJ9.r3Oom8qVEAd1ceMMWibrMNsgu0DPgz-IG13MAzB-o5s"
}
```

### logout.cgi

Logs out the user.

Response:
```json
{}
```

### reboot.cgi

Triggers a reboot.

If *reset_config* is set to 1 in the query, it resets the default configuration after rebooting.
If *bootloader* is 1, it reboots the unit to the bootloader.

Query (optional):
```json
{
  "reset_config": 1,
  "bootloader": 1
}
```
Response:
```json
{}
```

### status.cgi

Returns openSPOT's current status.

*status* can be:
  - 0: Standby
  - 1: In call
  - 2: Connector not set
  - 3: Connector connecting
  - 4: Modem initializing
  - 5: Modem disconnected
  - 6: Modem HW/SW version mismatch
  - 7: Modem firmware upgrade in progress

*rssi_tc0_values_dbm* and *rssi_tc1_values_dbm* contain RSSI values since
the last call of *status.cgi*. If the current modem mode is non-TDMA, then
ignore *rssi_tc1_values_dbm*.

*dejitter_buf_tc0_pkts* and *dejitter_buf_tc1_pkts* contain dejitter buffer
packet count values since the last call of *status.cgi*. If the current modem
mode is non-TDMA, then ignore *dejitter_buf_tc1_pkts*.

*ber_tc0_values* and *ber_tc1_values* contain the count of erroneous bits
since the last call of *status.cgi*. If the current modem mode is non-TDMA,
then ignore *ber_tc1_values*.

The packet and byte UDP traffic counters are monotonically increasing 32 bit values.

*callinfo* contains an array of call info structures. The first element of the
structure is the destination callsign (or DMR ID as a string), the second element
is the source callsign, and the third is the callinfo struct type. This is a 16-bit
value, the meaning of the bits:

- 0th bit (LSB): 0 - voice call, 1 - data call
- 1st bit: 0 - call from net, 1 - call from modem/openSPOT
- 2nd bit: 0 - group call, 1 - private call
- 3rd bit: 0 - call end, 1 - call start
- 4th bit: 1 - call started with late entry
- 5th bit: 1 - call ended with timeout
- 6th bit: 0 - TS1, 1 - TS2
- 7th bit: 1 - received a talker alias
- 8th bit: 1 - C4FM DN (VD) mode 1, 0 - C4FM DN (VD) mode 2
- 9th bit: 1 - C4FM wide voice mode, 0 - C4FM digital narrow mode
- 10th bit: 1 - DMR call
- 11th bit: 1 - C4FM call
- 12th bit: 1 - DSTAR call

Other bits are currently unused.

*connector* is the active connector ID (see connector.cgi's description for
connector ID names).

*primary* is 1 if the primary server is used by the current connector, and
it's 0 if the backup server is used.

*timeout_cp* is the config profile ID to switch on *timeout_cp_sec* timeout.
*timeout_null_sec* is the null connector switch timeout.

Response:
```json
{
  "status": 0,
  "connector": 2,
  "timeout_cp": 0,
  "timeout_cp_sec": 0,
  "timeout_null_sec": 0,
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
  "tx_bytes": 14421,
  "connected_to": "DCS001 A",
  "primary": 1,
  "callinfo": [{"9","2161005",0}, {"9","2161005",1032}]
}
```

### status-dmrsms.cgi

Returns current status of DMR SMS sending, if the modem is in DMR mode.
Otherwise it returns 400 Bad Request. It also handles SMS sending.

Call types: 0 - private, 1 - group.
Format IDs: 0 - ETSI, 1 - UDP, 2 - UDP/Chinese. See user manual for more info.
If a new message is received, *rx_msg_valid* is 1.
Setting a new *send_srcid* in the query overwrites the *default_srcid*.
If *send_to_modem* is 0, the SMS will be sent to the currently active connector.
If *intercept_net_msgs* is 1, then SMS messages coming from the network to
the *default_srcid* will be processed.
If *only_save* is 1, the SMS will not get sent, only the *send_srcid* and
*intercept_net* settings will be stored.

Messages are in hexadecimal UTF16BE format. Example: "BEER" = "0042004500450052"
Max. message length which can be sent is currently 75 UTF16BE characters (150
hex char pairs).

Query (optional):
```json
{
  "only_save": 0,
  "intercept_net_msgs": 0,
  "send_dstid": 2161005,
  "send_calltype": 0,
  "send_srcid": 9998,
  "send_format": 0,
  "send_tdma_channel": 0,
  "send_to_modem": 0,
  "send_msg": "0042004500450052"
}
```

Response:
```json
{
  "default_srcid": 9998,
  "intercept_net_msgs": 0,
  "send_ongoing": 0,
  "send_success": 1,
  "send_fail": 0,
  "rx_msg_valid": 0,
  "rx_msg_srcid": 1234,
  "rx_msg_dstid": 9998,
  "rx_msg_calltype": 0,
  "rx_msg_format": 0,
  "rx_msg_from_modem": 0,
  "rx_msg": "0042004500450052"
}
```

### status-srfipconnserver.cgi

Returns the SharkRF IP Connector Server's current status, if it's the
active connector. Otherwise it'll return 400 Bad Request.

*client_connected* is 1 if a client is connected.

Response:
```json
{
  "client_connected": 0,
  "client_id": 1234,
  "client_callsign": ""
}
```

### connector.cgi

openSPOT's active connector query (GET)/change (POST).

Valid connector IDs:

- 0: No connector set.
- 1: DMRplus
- 2: Homebrew
- 3: TS repeat
- 4: DCS/XLX
- 5: FCS
- 6: SharkRF IP Connector Client
- 7: SharkRF IP Connector Server
- 8: DMR demodulation mode auto calibration
- 9: REF/XRF
- 10: YSFReflector

Query (optional):
```json
{
  "connector": 0
}
```
Response:
```json
{
  "connector": 0
}
```

### connectorsettings.cgi

Connector other settings query (GET)/change (POST). Returns currently
active settings. *rxtimeout_sec* is the change to null connector timeout.

Query (optional):
```json
{
  "rxtimeout_sec": 0
}
```
Response:
```json
{
  "rxtimeout_sec": 0
}
```

### nullsettings.cgi

Null connector settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000
}
```

### dmrplussettings.cgi

DMRplus connector settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
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
Response:
```json
{
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

Homebrew connector settings query (GET)/change (POST). Returns currently
active settings.

Valid *autocon_calltype*, *c4fm_dstcalltype* and *reroute_calltype* values:
0 - group call, 1 - private call.

For more information about the MMDVM options field, see
[this](https://github.com/g4klx/MMDVMHost/blob/master/DMRplus_startup_options.md) description.

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 62030,
  "mmdvm_mode": 0,
  "mmdvm_options": "",
  "callsign": "",
  "password": "",
  "b_server_host": "",
  "b_port": 62030,
  "b_password": "",
  "b_toggle_timeout_sec": 60,
  "repeater_id": 901234,
  "autocon_id": 4771,
  "autocon_calltype": 0,
  "autocon_tdma_channel": 0,
  "autocon_interval_sec": 500,
  "autocon_discon": 0,
  "dmo_tdma_channel": 0,
  "c4fm_dstid": 9,
  "c4fm_dstcalltype": 0,
  "reroute_id": 9990,
  "reroute_calltype": 1,
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 62030,
  "mmdvm_mode": 0,
  "mmdvm_options": "",
  "callsign": "",
  "password": "",
  "b_server_host": "",
  "b_port": 62030,
  "b_password": "",
  "b_toggle_timeout_sec": 60,
  "repeater_id": 901234,
  "autocon_id": 4771,
  "autocon_tdma_channel": 0,
  "autocon_interval_sec": 500,
  "autocon_discon": 0,
  "dmo_tdma_channel": 0,
  "c4fm_dstid": 9,
  "c4fm_dstcalltype": 0,
  "reroute_id": 9990,
  "reroute_calltype": 1,
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30
}
```

### dcsxlxsettings.cgi

DCS/XLX connector settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "A",
  "reflector": "DCS025",
  "remote_module": "Z",
  "rx_timeout_sec": 30
}
```
Response:
```json
{
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

### refxrfsettings.cgi

REF/XRF connector settings query (GET)/change (POST). Returns currently
active settings.

*allow_g* is the "always allow module G to modem" setting.

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "D",
  "reflector": "REF001",
  "remote_module": "C",
  "allow_g": 1,
  "rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "D",
  "reflector": "REF001",
  "remote_module": "C",
  "allow_g": 1,
  "rx_timeout_sec": 1
}
```

### fcssettings.cgi

FCS connector settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "server_host": "fcs001.xreflector.net",
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "port": 12345,
  "callsign": "",
  "ccs7_id": 2161005,
  "reflector": "FCS001",
  "room_number": 25,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 30
}
```
Response:
```json
{
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

### ysfrefsettings.cgi

YSFReflector connector settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "server_host": "",
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "port": 42000,
  "callsign": "",
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "",
  "port": 42000,
  "callsign": "",
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10
}
```

### srfipconnclientsettings.cgi

SharkRF IP Connector Client settings query (GET)/change (POST).
Returns currently active settings.

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "192.168.3.120",
  "port": 65100,
  "id": 2161005,
  "password": "abcdefgh",
  "callsign": "",
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "server_host": "192.168.3.120",
  "port": 65100,
  "id": 2161005,
  "password": "abcdefgh",
  "callsign": "",
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30
}
```

### srfipconnserversettings.cgi

SharkRF IP Connector Server settings query (GET)/change (POST).
Returns currently active settings.

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "port": 65100,
  "password": "abcdefgh",
  "rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "port": 65100,
  "password": "abcdefgh",
  "rx_timeout_sec": 30
}
```

### dmrautocal.cgi

You can periodically query (GET) this CGI to get current DMR
demodulation mode auto calibration status. Also you can change
this connector's modem RX/TX frequencies. When the frequencies
are changed, the connector is restarted.

State can be:

- 0: idle
- 1: calibrating
- 2: finished, result available
- 3: modem is not in DMR mode

Progress is in percent. Result is the auto calibrated demodulation
mode (0 - A, 1 - B, 2 - C, etc.)

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "state": 1,
  "progress": 56,
  "result": 2
}
```

### c4fmautocal.cgi

You can periodically query (GET) this CGI to get current C4FM
demodulation mode auto calibration status. Also you can change
this connector's modem RX/TX frequencies. When the frequencies
are changed, the connector is restarted.

State can be:

- 0: idle
- 1: calibrating
- 2: finished, result available
- 3: modem is not in C4FM half deviation mode

Progress is in percent. Result is the auto calibrated demodulation
mode (0 - A, 1 - B, 2 - C, etc.)

Query (optional):
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
}
```
Response:
```json
{
  "rx_freq": 436000000,
  "tx_freq": 436000000,
  "state": 1,
  "progress": 56,
  "result": 2
}
```

### info.cgi

Allows you to query (GET) general info about the device.
*blver* is the bootloader version. *uid* is the device unique ID
in hexadecimal.
*uptime* is in seconds. *locked_to_country* is set to the country's
ISO code if there's a lock active on the current device.

Response:
```json
{
  "hwver": "1.0",
  "locked_to_country": "",
  "swver": "0001",
  "subver": "433",
  "blver": "0001",
  "uptime": 123,
  "mac": "FE:28:00:00:00:FA",
  "uid": "abcdef"
}
```

### cpsettings.cgi

Config profile settings query (GET)/change (POST). Returns currently
active settings.
*reboot* is 1 if the active config profile has been changed and openSPOT
is rebooting. *active_cp_initialized* is 1 if the currently active config
profile is initialized. The array *cp_names* contain each config profile's
name. *timeoutchange_cp* is the profile number which should be activated
after *rxtimeout_sec* seconds.

Query (optional):
```json
{
  "active_cp": 0,
  "timeoutchange_cp": 0,
  "rxtimeout_sec": 0,
  "active_cp_name": "default",
  "active_cp_copyto": 0
}
```
Response:
```json
{
  "reboot": 0,
  "active_cp": 0,
  "timeoutchange_cp": 0,
  "rxtimeout_sec": 0,
  "active_cp_hostname": "openspot",
  "active_cp_initialized": 1,
  "cp_names": ["default", "", "", "", ""]
}
```

### config-export.cgi

If you want to export the active config profile settings, you can POST
a query to this CGI. You can request settings in chunks. The number of
available chunks is in *chunk_count* (this can also be requested with a
GET query). Each chunk is represented in string of hexadecimal character
pairs. *config_size* is the count of valid config bytes in the file.
The whole config's CRC is returned in *config_crc*. Note that *chunk*
data in this example is truncated.

Query (optional):
```json
{
  "chunk_nr": 0
}
```
Response:
```json
{
  "config_size": 1458,
  "config_crc": "a9fd8832",
  "chunk_count": 3,
  "chunk_nr": 0,
  "chunk": "982443abcdef"
}
```

### config-import.cgi

If you want to import the active config profile settings, you can POST
a query to this CGI.

*config_size* is the count of valid config bytes in the file.
*chunk_size* is the length which this CGI waits *chunk* data in string
of hexadecimal character pairs.
*chunk_count* is the total number of *chunks* needed for a successful import.
*active_cp_hostname* is the hostname of openSPOT. This is always returned
because it may be changed after a successful import - in this case the caller
knows on what address openSPOT will start listening on after the device
reboots. Note that *chunk* data in this example is truncated.

*status* can be the following:

- 0: CONFIGAREA_IMPORT_STATUS_INITIALIZED
- 1: CONFIGAREA_IMPORT_STATUS_NEED_MORE_CHUNKS
- 2: CONFIGAREA_IMPORT_STATUS_SUCCESS
- 3: CONFIGAREA_IMPORT_STATUS_STRUCT_VERSION_MISMATCH
- 4: CONFIGAREA_IMPORT_STATUS_CRC_ERR
- 5: CONFIGAREA_IMPORT_STATUS_INVALID_FILE
- 6: CONFIGAREA_IMPORT_STATUS_FAIL

*status* will always be 0 when calling this CGI without query parameters.

Query (optional):
```json
{
  "config_size": 1458,
  "config_crc": "a9fd8832",
  "chunk_nr": 0,
  "chunk": "982443abcdef"
}
```
Response:
```json
{
  "chunk_size": 1024,
  "chunk_count": 3,
  "active_cp_hostname": "openspot",
  "status": 0
}
```

### passwordsettings.cgi

Allows you to change openSPOT's password.

Query:
```json
{
  "password": "openspot",
}
```
Response:
```json
{}
```

### netsettings.cgi

Network settings query (GET)/change (POST). Returns currently active settings.

*ip_config_mode* can be:
  - 0: DHCP
  - 1: DHCP with auto IP
  - 2: Auto IP
  - 3: Static IP

Query (optional):
```json
{
  "ip_config_mode": 0,
  "hostname": "openspot",
  "static_ip": "192.168.1.99",
  "static_mask": "255.255.255.0",
  "static_gw": "192.168.1.1",
  "static_dns1": "8.8.8.8",
  "static_dns2": "8.8.4.4",
  "usestaticdns": 1,
  "dejitter_queue_msec": 130
}
```
Response:
```json
{
  "ip_config_mode": 0,
  "hostname": "openspot",
  "static_ip": "192.168.1.99",
  "static_mask": "255.255.255.0",
  "static_gw": "192.168.1.1",
  "static_dns1": "8.8.8.8",
  "static_dns2": "8.8.4.4",
  "usestaticdns": 1,
  "dejitter_queue_msec": 130
}
```

### spksettings.cgi

Voice announcement settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "enabled": 1,
  "shortbm": 0,
  "remote_only": 0,
  "host": "spk.sharkrf.com",
  "port": 65200,
  "src_dmr_id": 9998,
  "prof_dmr_id": 9000,
  "con_dmr_id": 9998,
  "ip_dmr_id": 9001
}
```
Response:
```json
{
  "enabled": 1,
  "shortbm": 0,
  "remote_only": 0,
  "host": "spk.sharkrf.com",
  "port": 65200,
  "src_dmr_id": 9998,
  "prof_dmr_id": 9000,
  "con_dmr_id": 9998,
  "ip_dmr_id": 9001
}
```

### locationsettings.cgi

Location settings query (GET)/change (POST). Returns currently
active settings. Country is the country code (2 chars) or the string "global".

Query (optional):
```json
{
  "country": "global",
  "latitude": "0.0",
  "longitude": "0.0",
  "height_agl": 0,
  "name": ""
}
```
Response:
```json
{
  "latitude": "0.0",
  "longitude": "0.0",
  "height_agl": 0,
  "name": ""
}
```

### dmrsettings.cgi

General DMR settings query (GET)/change (POST). Returns currently
active settings.

If *no_inband* is set to 1, in-band DMR data like GPS pos. or talker alias
will not be sent to the modem.

Query (optional):
```json
{
  "force_talker_alias": "abcd",
  "no_inband": 0,
  "cc": 1,
  "echo_id": 9999,
  "default_c4fm_id": 0,
  "transmit_idle_in_idle_tx_tdma_channel": 1
}
```
Response:
```json
{
  "force_talker_alias": "abcd",
  "no_inband": 0,
  "cc": 1,
  "echo_id": 9999,
  "default_c4fm_id": 0,
  "transmit_idle_in_idle_tx_tdma_channel": 1
}
```

### dstarsettings.cgi

General D-STAR settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "echo_callsign": "       E",
  "transmit_rx_confirmation": 1
}
```
Response:
```json
{
  "echo_callsign": "       E",
  "transmit_rx_confirmation": 1
}
```

### c4fmsettings.cgi

General C4FM settings query (GET)/change (POST). Returns currently
active settings. *dmr_def_cs* is the default C4FM callsign for DMR calls.

If *only_rx_with_sql_code_en* is 1, then the modem only processes C4FM
calls with SQL code *only_rx_with_sql_code*.

If *force_sql_code_to_modem_en* is 1, all C4FM frames sent to the
modem will have SQL code *force_sql_code_to_modem* set.

If *force_sql_code_to_net_en* is 1, all C4FM frames sent to the
network will have SQL code *force_sql_code_to_net* set.

Query (optional):
```json
{
  "dtmf_automute_cmds": 1,
  "dtmf_pcode": "*",
  "dtmf_gcode": "#",
  "transmit_rx_confirmation": 1,
  "dmr_def_cs": "",
  "only_rx_with_sql_code_en": 0,
  "only_rx_with_sql_code": 123,
  "force_sql_code_to_modem_en": 0,
  "force_sql_code_to_modem": 123,
  "force_sql_code_to_net_en": 0,
  "force_sql_code_to_net": 123
}
```
Response:
```json
{
  "dtmf_automute_cmds": 1,
  "dtmf_pcode": "*",
  "dtmf_gcode": "#",
  "transmit_rx_confirmation": 1,
  "dmr_def_cs": "",
  "only_rx_with_sql_code_en": 0,
  "only_rx_with_sql_code": 123,
  "force_sql_code_to_modem_en": 0,
  "force_sql_code_to_modem": 123,
  "force_sql_code_to_net_en": 0,
  "force_sql_code_to_net": 123
}
```

### locksettings.cgi

Callsign/CCS7 ID lock settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "id1": 2161005,
  "callsign1": "HA2NON",
  "id2": 2161006,
  "callsign2": "HG1MA",
  "id3": 0,
  "callsign3": ""
}
```
Response:
```json
{
  "id1": 2161005,
  "callsign1": "HA2NON",
  "id2": 2161006,
  "callsign2": "HG1MA",
  "id3": 0,
  "callsign3": ""
}
```

### modemfreq.cgi

If you want to change the current RX, TX frequency or TX power
without reinitializing the modem, you can POST a query to this CGI.
Returns currently active settings. *modem_init_delay_ms* is the time needed
for the modem to calibrate and initialize.

DMR demodulation mode values: 0 - A, 1 - B, 2 - C etc.

Query (optional):
```json
{
  "rx_frequency": 433450000,
  "dmr_demodmode": 0,
  "tx_frequency": 433450000,
  "tx_power_percent": 100
}
```
Response:
```json
{
  "modem_init_delay_ms": 100,
  "rx_frequency": 433450000,
  "dmr_demodmode": 0,
  "tx_frequency": 433450000,
  "tx_power_percent": 100
}
```

### modemcwid.cgi

Modem CW ID settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "cwid": "HA2NON",
  "wpm": 25,
  "interval_sec": 600,
  "delay_sec": 30
}
```
Response:
```json
{
  "cwid": "HA2NON",
  "wpm": 25,
  "interval_sec": 600,
  "delay_sec": 30
}
```

### modemmode.cgi

Current modem mode query (GET)/change (POST). Returns currently
active settings. *modem_init_delay_ms* is the time needed for
the modem to calibrate and initialize.

- *mode* can be:
  - 0: Idle
  - 1: Raw
  - 2: DMR
  - 3: D-STAR
  - 4: C4FM
  - 5: C4FM half deviation mode

- *submode* can be:
  - 0: No submode set
  - 1: DMR Hotspot
  - 2: DMR MS
  - 3: DMR BS

Query (optional):
```json
{
  "mode": 0,
  "submode": 0
}
```
Response:
```json
{
  "modem_init_delay_ms": 3500,
  "mode": 0,
  "submode": 0
}
```

### modemmodulation.cgi

Modem modulation mode query (GET)/change (POST). Returns currently
active settings. *modem_init_delay_ms* is the time needed for the
modem to calibrate and initialize.

- *modulation_mode* can be:
  - 0: 2FSK
  - 1: 2FSK Raised Cosine
  - 2: 4FSK
  - 3: 4FSK Raised Cosine

Query (optional):
```json
{
  "modulation_mode": 0,
  "bitrate": 9600,
  "inner_deviation_hz": 648
}
```
Response:
```json
{
  "modem_init_delay_ms": 3500,
  "modulation_mode": 0,
  "bitrate": 9600,
  "inner_deviation_hz": 648
}
```

### modempacket.cgi

Modem packet settings query (GET)/change (POST). Returns currently
active settings. *modem_init_delay_ms* is the time needed for the
modem to calibrate and initialize.

Query (optional):
```json
{
  "packet_size_in_bits": 288,
  "sync_word_length_in_bits": 24,
  "sync_word_pos_in_packet_in_bits": 0,
  "sync_word_count": 1,
  "sync_word1": "4f5de8",
  "sync_word2": "445566"
}
```
Response:
```json
{
  "modem_init_delay_ms": 3500,
  "packet_size_in_bits": 288,
  "sync_word_length_in_bits": 24,
  "sync_word_pos_in_packet_in_bits": 0,
  "sync_word_count": 1,
  "sync_word1": "4f5de8",
  "sync_word2": "445566"
}
```

### modemtdma.cgi

Modem TDMA settings query (GET)/change (POST). Returns currently
active settings. *modem_init_delay_ms* is the time needed for the
modem to calibrate and initialize.

Query (optional):
```json
{
  "tdma_enabled": 1,
  "tdma_pit_calibration_wait_for_packets_num": 2,
  "tdma_pit_calibration_compensation_multiplier": 1.2,
  "tdma_needed_sync_frames_for_tdma_channel_be_valid": 1
}
```
Response:
```json
{
  "modem_init_delay_ms": 3500,
  "tdma_enabled": 1,
  "tdma_pit_calibration_wait_for_packets_num": 2,
  "tdma_pit_calibration_compensation_multiplier": 1.2,
  "tdma_needed_sync_frames_for_tdma_channel_be_valid": 1
}
```

### modemcal.cgi

Modem calibration settings query (GET)/change (POST). Returns currently
active settings. *modem_init_delay_ms* is the time needed for the modem
to calibrate and initialize.

Query (optional):
```json
{
  "auto_calibration": 1,
  "recalibrate_temp_diff_in_celsius": 10,
  "recalibrate_rf_ic_temp_diff_in_celsius": 10,
  "temp_read_interval_in_sec": 10,
  "temp_read_delay_in_sec_after_sync_lost": 10,
  "quick_calibrate_delay_in_sec_after_sync_lost": 10
}
```
Response:
```json
{
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

Other modem settings query (GET)/change (POST). Returns currently
active settings. *modem_init_delay_ms* is the time needed for the
modem to calibrate and initialize.

*agc_auto* and *external_vco* fields are booleans.

Query (optional):
```json
{
  "rssi_avg_sample_count": 5,
  "bclo_dbm": -80,
  "high_gain_low_linearity": 0,
  "agc_auto": 0,
  "agc_low_threshold_dbm": -50,
  "agc_high_threshold_dbm": -80,
  "external_vco": 0,
  "call_hang_time_ms": 3000
}
```
Response:
```json
{
  "modem_init_delay_ms": 3500,
  "rssi_avg_sample_count": 5,
  "bclo_dbm": -80,
  "high_gain_low_linearity": 0,
  "agc_auto": 0,
  "agc_low_threshold_dbm": -50,
  "agc_high_threshold_dbm": -80,
  "external_vco": 0,
  "call_hang_time_ms": 3000
}
```

### quickcall.cgi

If you want to request a quick DMR call, you can POST a query to this CGI.

Valid *call_type* values: 0 - group call, 1 - private call.

Query (optional):
```json
{
  "dst_id": 4000,
  "call_type": 0,
  "tdma_channel": 0
}
```
Response:
```json
{}
```

### bmmsettings.cgi

BrandMeister API settings query (GET)/change (POST). Returns currently
active settings.

Query (optional):
```json
{
  "apikey": "abcdef"
}
```
Response:
```json
{
  "apikey": "abcdef"
}
```
