Multi-Streamer -- Remote Control Protocol
=========================================

In the following you find an overview about the main concept and technical details how each Multi-Streamer instance (both the command line and the graphical version) can be observed as well as manipulated from a distant PC/server via a designated protocol. This protocol consists of special messages and timeout specifications which both are part of each running Multi-Streamer instance.

# 1. Core concept
* **Client/Server**: The remote control protocol is based  on a classical client-server architecture. The role of a **client** is to observe/control the resources (devices, streams, statistics, ..) located on the **server** side during a remote control season. Each of these season has a defined start and end time point. But there can always be multiple seasons running so that a server instance handles multiple clients at the same time while keeping its internal data always in a consistent state.
* **Auto Detection**: All running server instances in a LAN can be detected by local potential clients in almost real-time. For this purpose, the remote control protocol contains so-called "beacon" messages which are periodically sent by each server instance and are catched by local clients. Such a message contains all neccessarry data to start a new remote control session with the corresponding server.
* **Periodic Reports**: As soon as a client has started successfully a new remote control session with a server, the server starts to send periodic reports towards the client containing an overview about all server-side available devices, input streams and corresponding output streams. With this data a client creates and maintains its own overview about the status of the reporting server.
* **Request/Response**: The remote control protocol supports "actions". For example, a client can instruct a server to **add** or **remove inputs**. The same is possible for outputs. Each of these actions consists of a **request** message, which is send from a client to the server, and a subsequent **response** message, wich is send from the server to the client in return. The request describes the desired action and the response transmits the action result from the server to the client. Subsequently, the periodic reports inform all clients about server internal changes.

# 2. Protocol Packets
* **Structure**: Each packet of the remote control protocol consists of a small proprietary header with a fixed size of different elements, followed by a Google Protobuf data block with variable length. The following gives a detailed overview about the general structure of each message.
 
<pre>

       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +---------------+---------------+-------------------------------+
      |       'V'     |      'S'      |             Type              |
      +---------------+---------------+-------------------------------+
      |    reserved   |    Version    |       Transaction Id          |
      +---------------+---------------+-------------------------------+
      |                                                               |
      +                                                               +
      |                                                               |
      +                           Sender Id                           +
      |                                                               |
      +                                                               +
      |                                                               |
      +-------------------------------+-------------------------------+
      |                      Overall Payload Size                     |
      +-------------------------------+-------------------------------+
      |                     Fragment Payload Offset                   |
      +-------------------------------+-------------------------------+
      |                      Fragment Payload Size                    |
      +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
      |                    Serialized Payload Data                    |
      +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
 </pre> 
 
## 2.1 Packet elements
* **V** and **S**: This is a "magic code" to identify a protocol related packet. Although this is not 100% safe, it is better than having nothing to differentiate the remote control protcol from other transmissions.
* **Type**: With this element the general message type is selected. Possible values range from 0 to 65535. 
* **Version**: Since a protocol may vary over the years, this message lement is needed to differentiate between the different versions of the applied protocol definition. The current version is 0x12, which reppresents version "1.2".
* **Transaction Id**: Whenever a client triggers an action a transaction in the server is started. Both the request and the response contain in this case the same transaction ID. Based on this, a client can for each received response identify the corresponding request itself sent before.
* **Sender Id**: This is a unique ID (typically a UUID) which exists for each sender in the network. Each message sender (both clients and servers) creates an own ID during its startup and use it for all outgoing messages. Based on this ID, a sender can be clearly identified. Also a returning sender or a sender visible via 2 or more network interfaces can be clearly identified as the same one.
* **Overall Payload Size**: A transmitted Google Protobuf message can be split into multiple network packets where each packet transpports one single message fragment. This value represents the overall size of the resulting Google Protobuf message.
* **Fragment Payload Offset**: To be able to reconstruct the original Google Protobuf message at receiver side each fragment needs be clearly identified within the overall message. For this purpose, this element contains the offset from the start in the original message.
* **Fragment Payload Size**: This element contains the size of the fragment and can be combined with the previous offset value in order to find the correct point where the transmitted fragment belongs to. With the help of these two values the original Google Protobuf message can be reconstructed and processed at receiver side.
* **Serialized Payload Data**: These bytes contain the actual fragment data which is used to reconstruct the original Google Protobuf message.


## 2.2 "Type"
Like all streaming products from Volkert Software also Multi-Streamer uses internally the core library VASE ("Video Audio Stream Environment"). Due to this fact, the supported values for the "type" element are split into general VASE related ones and specialized Multi-Streamer specific ones. The boundary value between both ranges is 0x8000. The following lists all suppported values. Further details of the corresponding Google Protobuf message can be found in the individual "*.proto" file.

## 2.2.1 VASE related values
<code>

        SERVER_BEACON                                  = 0x01
        CLIENT_ALIVE                                   = 0x02
        
        /**********************
         * REPORTS
         **********************/
        REPORT_SYSTEM                                  = 0x11
        REPORT_SYSTEM_VOLATILES                        = 0x12
        REPORT_DEVICES                                 = 0x13
        REPORT_DEVICES_VOLATILES                       = 0x14
        REPORT_STREAMS                                 = 0x15
        REPORT_STREAMS_VOLATILES                       = 0x16
        
        /****************
         * LOGIN
         ****************/
        LOGIN                                          = 0x23
        LOGIN_RESPONSE                                 = 0x24
        
        /**********************
         * FILE SYSTEM ACCESS
         **********************/
        FS_EXISTS                                      = 0x31
        FS_EXISTS_RESPONSE                             = 0x32
        FS_GET_WORKING_DIRECTORY                       = 0x33
        FS_GET_WORKING_DIRECTORY_RESPONSE              = 0x34
        FS_BROWSE_DIRECTORY                            = 0x35
        FS_BROWSE_DIRECTORY_RESPONSE                   = 0x36
        FS_CREATE_DIRECTORY                            = 0x37
        FS_CREATE_DIRECTORY_RESPONSE                   = 0x38
        FS_GET_PARENT_DIRECTORY                        = 0x39
        FS_GET_PARENT_DIRECTORY_RESPONSE               = 0x3A
        FS_GET_DESKTOP_DIRECTORY                       = 0x3B
        FS_GET_DESKTOP_DIRECTORY_RESPONSE              = 0x3C
        
        /**********************
         * ADD INPUTS
         **********************/
        ADD_INPUT_FOR_NETWORK                          = 0x101
        ADD_INPUT_FOR_URL                              = 0x102
        ADD_INPUT_FOR_LIVE_DEVICE_CAPTURING            = 0x103
        ADD_INPUT_FOR_NDI_STREAM                       = 0x105
        ADD_INPUT_FOR_DECKLINK_DEVICE_CAPTURING        = 0x106
        ADD_INPUT_FOR_CORVIDKONA_DEVICE_CAPTURING      = 0x107
        REMOVE_INPUT                                   = 0x121
        // <---- response
        ADD_INPUT_RESPONSE                             = 0x151
        REMOVE_INPUT_RESPONSE                          = 0x152
        
        /**********************
         * SET INPUT ATTRBUTES
         **********************/
        INPUT_DATA_FOWARDING_SET_MUTED                 = 0x171
        INPUT_DATA_FOWARDING_SET_PAUSED                = 0x172
        INPUT_SEEK_TO_POSITON                          = 0x173
        INPUT_SEEK_TO_POSITON_RESPONSE                 = 0x174
        
        /**********************
         * PLAYBACK ATTRBUTES
         **********************/
        INPUT_PLAYBACK_SET_PAUSED                      = 0x181
        INPUT_PLAYBACK_SET_PREBUFFER_AVOID_STALLING    = 0x182
        INPUT_PLAYBACK_SET_PREBUFFER_TIME              = 0x183
        INPUT_PLAYBACK_MEDIA_PACKET                    = 0x190
        INPUT_PLAYBACK_SUBSCRIBE_TO_MEDIA_PACKETS      = 0x191
        INPUT_PLAYBACK_UNSUBSCRIBE_FROM_MEDIA_PACKETS  = 0x192
        
        /**********************
         * ADD OUTPUTS
         **********************/
        ADD_OUTPUT_FOR_NETWORK                         = 0x201
        ADD_OUTPUT_FOR_FILE_RECORDING                  = 0x202
        ADD_OUTPUT_FOR_NDI_STREAM                      = 0x205
        ADD_OUTPUT_FOR_DECKLINK_DEVICE_PLAYBACK        = 0x206
        ADD_OUTPUT_FOR_CORVIDKONA_DEVICE_PLAYBACK      = 0x207
        REMOVE_OUTPUT                                  = 0x221
        // <---- response
        ADD_OUTPUT_RESPONSE                            = 0x251
        REMOVE_OUTPUT_RESPONSE                         = 0x252
        
        /**********************
         * SET OUTPUT ATTRBUTES
         **********************/
        OUTPUT_DATA_FOWARDING_SET_PAUSED               = 0x272
        
        /**********************
         * ADD SERVERS
         **********************/
        ADD_SERVER                                     = 0x301
        REMOVE_SERVER                                  = 0x321
        // <---- response
        ADD_SERVER_RESPONSE                            = 0x351
        REMOVE_SERVER_RESPONSE                         = 0x352
        
        /**********************
         * SERVER PATHS
         **********************/
        PUBLISH_SERVER_PATH                            = 0x401
        REVOKE_SERVER_PATH                             = 0x421
        // <---- response
        PUBLISH_SERVER_PATH_RESPONSE                   = 0x451
        REVOKE_SERVER_PATH_RESPONSE                    = 0x452
        
        /**********************
         * SERVER SESSIONS
         **********************/
        REMOVE_SERVER_SESSION                          = 0x501
        // <---- response
        REMOVE_SERVER_SESSION_RESPONSE                 = 0x551
</code>

## 2.2.2 Multi-Streamer specific values

<code>

        /**********************
         * REPORTS: LOCAL -> RPC
         **********************/
        ACCOUNT_REPORT                                 = 0x8001 // data about currently selected account
        LICENSE_REPORT                                 = 0x8002 // data about currently selected license

        /**********************
         * LICENSE CONFIGURATION: RPC -> LOCAL
         **********************/
        LICENSE_ACTIVATE                               = 0x8011
        LICENSE_SELECT                                 = 0x8012
        LICENSE_SET_NAME                               = 0x8013
        LICENSE_SET_COMMENT                            = 0x8014

        /**********************
         * ACCOUNT SELECTION: RPC -> LOCAL
         **********************/
        ACCOUNT_LOGIN                                  = 0x8021
        ACCOUNT_LOGIN_RESULT                           = 0x8022
        ACCOUNT_LOGOUT                                 = 0x8023
        
        /**********************
         * ACCOUNT CREATION: RPC -> LOCAL
         **********************/
        ACCOUNT_REQUEST_CREATE_CODE                    = 0x8031
        ACCOUNT_CREATE                                 = 0x8032
        
        /**********************
         * ACCOUNT CONFIGURATION: RPC -> LOCAL
         **********************/
        ACCOUNT_EDIT_USER_DETAILS                      = 0x8041
        ACCOUNT_CHANGE_PASSWORD                        = 0x8042
        
        /**********************
         * ACCOUNT RECOVERY: RPC -> LOCAL
         **********************/
        ACCOUNT_REQUEST_PASSWORD_RECOVERY_CODE         = 0x8051
        ACCOUNT_RECOVERY_PASSWORD                      = 0x8052
        
        REQUEST_RESULT                                 = 0x8100
</code>

# 3. Detecting a server instance in the LAN
The following list contains a chronologically sorted list of the needed steps:
* **server announces** its service: Each server instance sends periodically a so-called **"beacon"** message. It is encapsulated in a protocol packet (see 2.) which is send via **UDP** from **source port 9110** to **destination port 59110** along **all local network interfaces** of the underlying machine. Such a packet is send both to the IPv4 broadcast address **255.255.255.255** and the IPv6 address **ff02::1**.
* **client receives** the announce: A typical client implementation listens on **all local network interfaces** for **UDP** packets which are sent to **destination port 59110** and destination IP address **255.255.255.255** or **ff02::1**. Each packet is parsed and the original **"beacon"** message gets extracted. The containing information (e.g., the management port at server side) can then be used for establishing a remote control session with the individual server.

# 4. Starting a remote control session
A new session from a client to a server is started from the client side as follows:
* **starting TCP connection**: For each remote control session a central TCP connection between client and server is needed. The client needs to keep the connection valid as long as the remote control session is used. For the initial TCP connect attempts the client needs to know the management port of the server (see 3.).
* **log in to the control service**: In order to have a usable remote control session, a client needs to transmit to the server the central service password to activate access permission to all server internal data and be enabled to control inputs as well as outputs. For this purpose, the client sends a **"login"** message containing the ppassword.
* **wait for response**: A server instance responds to each **"login"** message with a **"login response"** message. The client needs to parse it and extract the result code (see appendix A for a list) from it.

# 5. Example: add a network input
In the following a chronological order of needed steps are listed to start an additional network input at server side:
* (optional) **detect the server**: see 3. for details of this step
* **start session**: see 4. for details of this step
* **send the request**: The client starts the action with a **"add network input"** message, which contains the following parameters:
  * **url**: This is a pure string containing the full address from where the stream should be pulled.
  * **network protocol parameters**: Additional parameters (e.g., SRT delay) can be given as key-value-pair list.
  * **demuxer format**: This value can either be "auto" or it can explicitly select a demuxer format.
  * **decoder settings**: The client can instruct the server to activate or ignore video/audio sub streams in the new network input. Moreover, the client can select if the server should use software or hardware based video decoding by selecting a gpu.
* **wait for response**: A server instance responds to each **"add network input"** message with a **"add input response"** message. The client needs to parse it and extract the result code (see appendix A for a list) from it.
* **process response**: The **"add input response"** message contains besides the result code also the UUID of the created input at server side (if the action was successful). This ID can later be used to trigger further actions (e.g., to add outputs to this created input).

# Appendix A: VASE result codes
The following list gives an overview about the most important result codes:
<code>

* **VASE_OKAY_RECALL_NEEDED (2)**: e.g., a connection attempt needs a retry
* **VASE_OKAY (1)**
* **VASE_ERR_UNDEFINED (-2)**: undefined error: error cause is undefined
* **VASE_ERR_LICENSE (-3)**: license error: the purchased VASE license doesn't contain this feature 
* **VASE_ERR_PARAMETER (-4)**: parameter error: one or more parameters are wrong 
* **VASE_ERR_TIMEOUT (-5)**: timeout: a timeout has occurred 
* **VASE_ERR_DUPLICATED_CALL (-6)**: duplicated call: a duplicated request has occurred 
* **VASE_ERR_MISSING_INIT (-7)**: missing initialization: one or more initialization calls are missing 
* **VASE_ERR_MISSING_LIBRARY (-8)**: missing library: one or more needed libraries could not be loaded dynamically during runtime 
* **VASE_ERR_UNSUPPORTED_CALL (-9)**: unsupported call: the call isn't supported with the given instance 
* **VASE_ERR_NET_CLIENT (-10)**: network client error: any network problem, e.g., a TCP connect failed 
* **VASE_ERR_NET_CLIENT_DNS_RESOLUTION (-11)**: network DNS resolution error: a given host name could not be resolved to a valid IP address 
* **VASE_ERR_NET_CLIENT_ROUTING (-12)**: network routing error: no route for the given IP address was found 
* **VASE_ERR_NET_CLIENT_NO_FREE_LOCAL_PORT (-13)**: network port allocation error: no free local port available 
* **VASE_ERR_NET_CLIENT_TLS (-14)**: network transport error: general TLS problem occured 
* **VASE_ERR_NET_CLIENT_ADDRESS_USED (-15)**: network socket error: the socket address (IP & port) is used 
* **VASE_ERR_NET_CLIENT_NETWORK_DOWN (-16)**: network error: the network is down 
* **VASE_ERR_NET_CLIENT_NETWORK_UNREACHABLE (-17)**: network error: the network is unreachable 
* **VASE_ERR_NET_CLIENT_PEER_RESET (-18)**: network transport error: the connection was reset by peer 
* **VASE_ERR_NET_CLIENT_TIMEOUT (-19)**: network transport error: the connection timed out 
* **VASE_ERR_NET_CLIENT_CONNECT_REFUSED (-20)**: network transport error: the connection attempt was refused 
* **VASE_ERR_NET_CLIENT_UNAUTHORIZED (-21)**: network transport error: the connection is unauthorized 
* **VASE_ERR_NET_CLIENT_PASS_WRONG (-22)**: network transport error: the connection password was wrong (e.g., for SRT connection or RTMP publishing) 
* **VASE_ERR_NET_CLIENT_PASS_STATE (-23)**: network transport error: the connection password was missing/unexpected (e.g., for SRT) 
* **VASE_ERR_NET_CLIENT_STREAMID_WRONG (-24)**: network transport error: the connection stream ID was wrong (only for SRT) 
* **VASE_ERR_NET_CLIENT_MESSAGE_STREAM_API_STATE (-25)**: network transport error: the connection attempt failed due to Message/Stream API collision (only for SRT)
* **VASE_ERR_NET_CLIENT_PASS_MISSING (-26)**: network transport error: the connection password was missing 
* **VASE_ERR_NET_CLIENT_PASS_UNEXPECTED (-27)**: network transport error: the connection password was unexpected 
* **VASE_ERR_NET_CLIENT_PORT_UNREACHABLE (-28)**: network transport error: the connection attempt failed due to unreachable remote port, e.g., no listener is running remotely
* **VASE_ERR_NET_CLIENT_PATH_UNAVAILABLE (-29)**: network transport error: the connection tried to access an unavailable path at the remote side 
* **VASE_ERR_NET_CLIENT_PORT_BINDING (-30)**: network transport error: the port binding failed (e.g., for SRT) 
* **VASE_ERR_NET_CLIENT_ABORT (-31)**: network transport error: an explicit abort was triggered 
* **VASE_ERR_NET_HIGHER_CLIENT_RTMP_HANDSHAKE_FAILED (-33)**: higher network client: RTMP handshake failed 
* **VASE_ERR_NET_CLIENT_SRT_BONDING_MODE_REFUSED (-39)**: network transport error: the used SRT bonding mode was refused 
* **VASE_ERR_NET_SERVER_MULTICAST_JOIN (-40)**: network multicast error: could not join multicast group 
* **VASE_ERR_NET_SERVER_HOST (-41)**: network host error: given server host doesn't belong to a locally valid IP address 
* **VASE_ERR_FILE_NOT_FOUND (-80)**: file access: the desired input file was not found 
* **VASE_ERR_FILE_READING (-81)**: file access: the desired file read was not possible 
* **VASE_ERR_FILE_WRITING (-82)**: file access: the desired file write was not possible 
* **VASE_ERR_DEVICE_ACCESS (-90)**: device access: there was a problem accessing the desired device 
* **VASE_ERR_DEVICE_ACCESS_PERMISSION (-91)**: device access: there was a problem accessing the desired device due to missing permission 
* **VASE_ERR_RPC_SERVER_RESPONSE_TIMEOUT (-201)**: RPC: the RPC server response timed out 
* **VASE_ERR_RPC_CLIENT_UNAUTHORIZED (-202)**: RPC: the RPC request is unauthorized 
* **VASE_ERR_RPC_CLIENT_LOGIN_DENIED (-203)**: RPC: the RPC login was denied 
* **VASE_ERR_RPC_CLIENT_UNKNOWN_OBJECT (-204)**: RPC: the RPC request referenced an unknown object 
* **VASE_ERR_SERIALIZATION_SPEC_DEPRECATED (-1000)**: a serialized data structure (a file) has used a too old spec.                                             
* **VASE_ERR_SERIALIZATION_WRONG_FORMAT (-1001)**
</code>

# Appendix B: (de-)muxer formats
VASE as well as Multi-Streamer supports the following formats for demuxing and muxing:
<code>

        UNKNOWN   = -1
        AUTO      = 0
        ADTS      = 1
        AVI       = 2
        FLV       = 3
        H264      = 4
        HEVC      = 5
        VVC       = 6
        HLS       = 7
        MATROSKA  = 8
        M3U       = 9
        M4V       = 10
        MOV       = 11
        MP3       = 12
        MP4       = 13
        MPEG      = 14
        MPEGTS    = 15
        MPEG_DASH = 16
        MXF       = 17
        WEBM      = 18
        WMV       = 19
</code>
