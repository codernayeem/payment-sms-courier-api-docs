Time for integrating mobile otp service.

Code
Meaning
202	SMS Submitted Successfully
1001	Invalid Number
1002	sender id not correct/sender id is disabled
1003	Please Required all fields/Contact Your System Administrator
1005	Internal Error
1006	Balance Validity Not Available
1007	Balance Insufficient
1011	User Id not found
1012	Masking SMS must be sent in Bengali
1013	Sender Id has not found Gateway by api key
1014	Sender Type Name not found using this sender by api key
1015	Sender Id has not found Any Valid Gateway by api key
1016	Sender Type Name Active Price Info not found by this sender id
1017	Sender Type Name Price Info not found by this sender id
1018	The Owner of this (username) Account is disabled
1019	The (sender type name) Price of this (username) Account is disabled
1020	The parent of this account is not found.
1021	The parent active (sender type name) price of this account is not found.
1031	Your Account Not Verified, Please Contact Administrator.
1032	ip Not whitelisted

POST API

Parameter Name, Meaning/Value, Required, Description
api_key,	API Key,	Yes,	API Key : <<<BULKSMSBD_API_KEY>>>
senderid,	Approved Sender ID,	Yes,	Sender ID : <<<BULKSMSBD_SENDER_ID>>>
number,	mobile number,	Yes,	Exp: 88017XXXXXXXX,88018XXXXXXXX,88019XXXXXXXX...
message,	SMS body,	Yes,	Please use url encoding to send some special characters like &, $, @ etc



                            

  type : "post",
  url : "http://bulksmsbd.net/api/smsapi",
  data : {
    "api_key" : "your api key",
    "senderid" : "sender id",
    "number" : "88016xxxxxxxx,88019xxxxxxxx",
    "message" : "your test sms content"
  }

OTP for user login, order/payment confirmation & other service. 
If error occurs, then let user know (dont show technical info)
