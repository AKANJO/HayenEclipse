# HayenEclipse
Hayen Eclipse Server/Client Frame description


LP pulse water metering modem.
# Data Frames. Ver 1.3

The Notification Data frame sent by modem is of the form:

Notification Frame: Modem to Server

______________________________________ ___________________________________________________
Field:       Frame_Start  Type  ModemID	 Status	Payload_Size	Payload  CRC16	 Frame_End  |
Size(Byte):  1 	          1  	 4     	   1 	    2         	   ……..	   2 	     1          |
Value:       0x7E						                                           CCITT	  0x7E      |
_______________________________________ __________________________________________________|

Status Field(1Byte) 
_______________________________________________
0	               1	         2	      3 4 5 6	7
Cover_Opened	SIM_Removed	Low_Power 	0	0	0	0	0
_______________________________________________

Long and short integer are stored with the little endian arrangement.
“Type” field will take “0” for DeviceInfo, “1” for PR7 Data and “2” for MBus Data.
Modem Status contains the following signal:

CRC16 is a of type CCITT (0x1021) calculated for all the frame except the frame End.
So far, we defined 3 types of payload: 


# 1-	DeviceInfo payload sent whenever modem starts (sent every startup), 
    that contains modem and device info parameters,

_______________________________________________
Time_Stamp	IMEI	Modem Type	SIM ID	IMSI	Version	IP_Adress	Local_Port
4           16   	16	         20	     16    	8     	16     	2 bytes
_______________________________________________

_______________________________________________
Reset_Count	Battery Voltage	Brand	Status	Signal Strength	Meter Type	Meter Count
2         	2             	1    	1      	1             	1         	1 byte
_______________________________________________


Meter Type: “0xFF” for MBus meter else for Pulse counter.  0x00 = K1:1, 0x01 = K1:10, 0x11 = K10:10,
0x02= K1:100, 0x12 = K10:100… etc

*************
# 2-	PR7 data payload: 
is an hourly sampling data. It is usually 24 samples per day (per transfer)
PR7 data payload

-----------------------------------------------
Sample count	         Sampling Data Frames
1 byte usually = 24	    24 frames = 17*24 bytes
----------------------------------------------- -
 

Data sample Frame (Fixed size 17 Bytes)

---------------------------------------------------------------------
Time Stamp	Forward Pulses	Reverse Pulses	Compensated Pulses	CTR
4 Bytes	     4 Bytes	      4 Bytes	        4 Bytes	            1 Byte
-------------------------------------------------------------------- -

CTR field

------------------------------------------------
0	      1	2	3	4	5	6	7
Passive	0	0	0	0	0	0	Tamper/Battery empty
----------------------------------------------- -


Usually:
Forward Pulses – Reverse Pulses = Compensated pulses. 
PR7/6 pulse generator contains 4 signals
Pulse1P indicate a real flow where Pulse1D indicate the direction of the flow, from those signals Modem counts Forward Pulses and reverse pulses,
Pulse2 counts only the compensated flow (Forward - Reverse) where Pulse2C indicates if there is compensation or not. I do not use this signal in the data.

*************
# 3-	MBus Data payload,
Modem could be attached to multiple MBus meters, each meter should be read every hour, so there are usually 24 samples for each meter per day (per transfer).
The following frame represents sampling data array for multiple MBus meters, the first field indicates the overall data samples contained in the frame; 

MBus Data payload
__________________________________________________ _________
Sample count	MBus Data sample 1	MBus Data sample 2	Etc …
1 Byte 	      About 64 bytes		  About 64 bytes
__________________________________________________ __________

MBus Data (non-fixed size)
_______________________________________________
Time Stamp	Meter ID	Length L	Data	   Spacer
4 Bytes	    1 byte	  1 Byte	  L bytes	  0X7C
_______________________________________________


*************
*************
# II. Server Response & Command
~After receiving of a data notification from the modem, server will respond by a UDP frame that contains a result of processing the notification and a command if user wants to perform a command to the modem. The response has the following format:
____________________________________________________________________________________
Start	Error Code	Accepted Samples	Time stamp	Cmd Size N	Command	  BCC	      End
0x7E	1 byte	      2 bytes	         4 bytes	   2 bytes	  N bytes	   1 byte	  0x7E
_____________________________________________________________________________________
Frame Start & End : 0x7E
Accepted Samples: is the number of correct received data samples, It is for future use.
Error code: indicate the error occurred during notification parsing, 

	sr_Err_None         =0   // OK
	sr_Err_Truncated    =1   // uncompleted notification
	sr_Err_Size         =2   // error in payload size or data size
	sr_Err_CRC16        =3   // wrong crc16

Command: is a char string, its size stored in the command size field. Multiple commands can be included by separating them by “\n” character.



Examples: will be shared later




# Appendix: The data structure definition
 #define SERVER_FRAME_VERSION 1.3

  *typedef struct {
    uint8_t CoverOpened:1;
    uint8_t SIMRemoved:1;
    uint8_t LowPower:1;
    //uint8_t Reserved:5;
  }TModemStatus;

typedef enum{
	sft_DeviceInfo =0
      ,sft_PR7Data    =1
      ,sft_MBusData   =2
} TServerFrameType;
	
//Notification Frame
typedef struct
{
	uint8_t      StartChar;                 // Frame Start
	TServerFrameType Type;                 // Frame Type
	uint32_t     ModemID;               // Modem Identification Number
	TModemStatus Status;
	uint8_t*     Payload;
	uint16_t     PayloadSize;              // Frame size in bytes
	uint16_t     CRC16;
	uint8_t      EndChar;                 // Frame End
} TServerFrame;

//DeviceInfo Notification
typedef struct 
{
	char         IMEI[16];
  char         ModemType[16];
	char         SIMID[20];
	char	      IMSI[16];
	char         Version[8];
	char         IP[16];
	uint16_t     LocalPort;
	uint16_t     ResetCount;
	uint16_t     BatteryVoltage;
  uint8_t      Brand;
	TModemStatus Status;           
	uint8_t      SignalStrength;   //CSQ
	uint8_t      MeterType;
	uint8_t      MeterCount;
	
	// etc...
} TDeviceInfoPayload;

typedef struct 
{
	uint32_t TimeStamp;   //sampling Time value
	uint32_t FPulse1;	  //Forward pulse counter
	uint32_t RPulse1;	  //Reverse Pulse counter
	uint32_t Pulse2;	  //OverAll (Compensated) Pulse counter
	uint8_t Tamper;	  //Tamper Flag
}TPR7Record;

typedef struct 
{
  uint8_t RecordCount;  //usually 24
  TPR7Record Record[…];
} TPR7Payload;

typedef struct 
{
	uint32_t TimeStamp;
	uint8_t MeterID;
	uint8_t Size;        // Size < 128-8
	uint8_t Buf[Size];   //TODO: later, we will configure the record size
       uint8_t Spacer;      //0x7C
}TMBusRecord;

typedef struct 
{
  uint8_t RecordCount;     //usually 24 for each meter
  TMBusRecord Record[…];   //only for one meter	
} TMBusPayload;


typedef enum{
	 sr_Err_None         =0
	,sr_Err_Truncated    =1
	,sr_Err_Size         =2
	,sr_Err_CRC16        =3
} TSrvResponseErr;


//Notification response & Command Frame
typedef struct
{
	uint8_t   Start;                 // Frame reference number
	TSrvResponseErr Error;                 // Error kind
	uint16_t  AcceptedSamples;        // how much samples are correct counting from start of the sample list
	uint16_t  TimeStamp;              // Server Time
	uint16_t  CmdSize;            // Req size in bytes
	uint8_t*  Cmd;
	uint8_t   BCC;                 // Block Check Character
	uint8_t   End;                 // Frame End = 0x7E

} TSrvResponse;

