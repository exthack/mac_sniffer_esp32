Documentation 

Before getting started I am including all the nesscary files 


This library is for the wifi driver, which will help us sniff around. Also, after sniffing, with the help of this library, we will be connecting to the wifi, opening our own web server, and sharing all the sniff mac addresses.

    #include <WiFi.h>
    #include <Wire.h>
    #include "esp_wifi.h"


This library will help us wrap our data and send it over to the server. This library can be downloaded from Audrio's Manage Library section.
    #include <Arduino_JSON.h>     

Now this is the last set of the library that we will include, and all the libraries are not available; you need to download them manually from Guthub.

This library open the webserver 
    #include <AsyncTCP.h> 
This library open the webserver 
    #include <ESPAsyncWebServer.h>

SPIFFS will be responsible for handling requests and response.(HTTP PROTOCOL)
    #include "SPIFFS.h"

Now let's define some function in `void setup()`
/* start Serial */
    Serial.begin(115200);
/* setup wifi */



WIFI_INIT_CONFIG_DEFAULT macro to initialize the configuration to default values. all configuration will be saved on `cfg` varaible


    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();

Don't try to print cfg you will end up in error.
----------------------------------------------------------------------------------

Initialize WiFi Allocate resource for WiFi driver, such as WiFi control structure, RX/TX buffer, WiFi NVS structure etc. This WiFi also starts WiFi task.
This API must be called before all other WiFi API can be called

    esp_wifi_init(&cfg);


Returns
    ESP_OK: succeed
    ESP_ERR_NO_MEM: out of memory
    others: refer to error code esp_err.h


Set the WiFi API configuration storage type.The default value is `WIFI_STORAGE_FLASH` But we will be setting up in the  `WIFI_STORAGE_RAM` For more storage you refer this article https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_wifi.html#_CPPv420esp_wifi_set_storage14wifi_storage_t

    esp_wifi_set_storage(WIFI_STORAGE_RAM);


Setting up the wifi in null mode There are various modes in wifi, such as station mode and AP mode. You can google it or refer to the expressif documentation.

    esp_wifi_set_mode(WIFI_MODE_NULL);

Now that we have all our configurations set, we are ready to start our wifi with these configurations. As discussed, this configuration is important; otherwise, if we directly call "esp_wifi_start()," the wifi mode will be WIFI_MODE_STA, which is a station mode, and we don't want that.

    esp_wifi_start();

Enable the promiscuous mode.It is boolen output is false - disable, true - enable
    esp_wifi_set_promiscuous(true);

We are interested in catching packets because whenever the client wants to connect to a wifi, he will connect to the wifi and send a packet. And that packet is in a management frame, and it is an unencrypted packet. In that frame, we will try to capture a mac address.To capture only packets we will need apply filter to promiscuous.

    esp_wifi_set_promiscuous_filter(&filt);

In the above code, I have given the parameter "&filt," which means that whenever I receive a packet, it gets triggered. But we need a better filter so that we only capture what we need and not more. Even if we capture more, you won't be able to read it, as all the packets will be encrypted in the air. For more, you can refer to how devices and WiFi communicate.

    const wifi_promiscuous_filter_t filt={
     .filter_mask=WIFI_PROMIS_FILTER_MASK_MGMT|WIFI_PROMIS_FILTER_MASK_DATA
    };

WIFI_PROMIS_FILTER_MASK_MGMT :- filter the packets with type of WIFI_PKT_MGMT
WIFI_PROMIS_FILTER_MASK_DATA :- filter the packets with type of WIFI_PKT_DATA

----------------------------------------------------------------------------------------

Now that we have set the filters for packet listening, whenever a packet is received, it will go through the filters, and now we need to register the RX callback function in promiscuous mode. Each time a filter packet is received, the registered callback function will be called. 

    esp_wifi_set_promiscuous_rx_cb(&sniffer);

In the above code, we have  a function that will be called whenever a packet is received. Now we need to create a function with the same name.

This is where packets end up after they get sniffed

    void sniffer(void* buf, wifi_promiscuous_pkt_type_t type) {
        wifi_promiscuous_pkt_t *p = (wifi_promiscuous_pkt_t*)buf;  //P will be our payload. 
        int len = p->rx_ctrl.sig_len;   /*Data or management payload. Length of payload is described by rx_ctrl.sig_len.Type of content determined by packet type argument of callback.*/

    }
You can also print; you will get to know the packet size.
    Serial.println(len);

Now comes the complicated part, which you will need to analyze.
Before procedding further we need to create a 2 typedef struct . So here we will use  type def struct to  store data . 
    typedef struct { // or this
        uint8_t mac[6];
    } __attribute__((packed)) MacAddr;

    typedef struct { // still dont know much about this
        int16_t fctl;
        int16_t duration;
        MacAddr da;
        MacAddr sa;
        MacAddr bssid;
        int16_t seqctl;
        unsigned char payload[];
    } __attribute__((packed)) WifiMgmtHdr;

Now let's continue with our function 
    void sniffer(void* buf, wifi_promiscuous_pkt_type_t type) {
        wifi_promiscuous_pkt_t *p = (wifi_promiscuous_pkt_t*)buf;  //P will be our payload. 
        int len = p->rx_ctrl.sig_len;   /*Data or management payload. Length of payload is described by rx_ctrl.sig_len.Type of content determined by packet type argument of callback.*/
        WifiMgmtHdr *wh = (WifiMgmtHdr*)p->payload;
        len -= sizeof(WifiMgmtHdr);
        if (len < 0){
            Serial.println("Receuved 0");
            return;
        }
    }


Now Let's define two variable of datatype String,
    String packet;
    String mac;
    int fctl = ntohs(wh->fctl); //IDK what this does . if anyone knows please let me know 

---------------------------------------------------------------

// This reads the first couple of bytes of the packet. This is where you can read the whole packet replaceing the `8+6+1` with `p->rx_ctrl.sig_len`

    for(int i=8;i<=8+6+1;i++){ 
        packet += String(p->payload[i],HEX);
    }

This removes the 'nibble' bits from the stat and end of the data we want. So we only get the  mac address. 

    for(int i=4;i<=15;i++){ 
        mac += packet[i];
       
    }
If you see mac address are usually in UPPERCASE so we will convert it to uppercase
    mac.toUpperCase();

Import Note :- 

Few Mac addresses are only captured with 1  missing numbers. I have noticed this on Mac addresses, which start at 0. You can analyze by printing mac after for loop  here how's
    for(int i=4;i<=15;i++){ 
        mac += packet[i];      
    }
     Serial.println(mac);

Even if you print the packet above, you will get the same output as on a Mac. The reason I used it was that sometimes packets might contain some extra, so to avoid that, I have to do so.


Now let's move to the trickiest part of the code. Before that, we need to create a 2-dimensional array.

    String maclist[64][3]   //It means 64 rows and 3 columns 
    int listcount = 0;
    String defaultTTL = "60";
Make sure you define these outside functions.

    int added = 0;
    for(int i=0;i<=63;i++){ 
        if(mac == maclist[i][0]){
            maclist[i][1] = defaultTTL;
            if(maclist[i][2] == "OFFLINE"){
                maclist[i][2] = "0";
            }
        added = 1;
    }
  

Now let's go line by line.
First we have declared a variable called added, which equals 0.
Then we are running a for loop to check if the MAC address has been added before.
So basically, we have not yet stored a Mac address in an array. But we have already defined an array, `maclist[64][3]`.
So we are checking `if (mac == maclist[i][0])`. If that mac address is present on that array, we are going to supply a default TTL, which is defined as `int 60` Why we are doing that, you will get to know later, but still, I will let you know TTL (time to leave ). which will be later used to know if the user is online or offline.

As you should know how 2 dimensional array works . so we have  `maclist[64][3]`

64 rows
    1st column is where we are going to store `Mac`
    2nd column, where we are going to store the `TTL` value
    3rd column, where we are going to store `status`

If our if condition confirm , our added value will be 1, and we will use this logic to create another if conditions. and after that, you will get to know this code.
    if(maclist[i][2] == "OFFLINE"){
        maclist[i][2] = "0";
    }

So let's think if Mac is not present in our Maclist array, so we need to create a logic so that it gets added to our array.

    if(added == 0){ // If its new. add it to the array.
        maclist[listcount][0] = mac;
        maclist[listcount][1] = defaultTTL;
        listcount ++;
        if(listcount >= 64){
            Serial.println("Too many addresses");
            listcount = 0;
        }
    }


This is pure self-explanatory; it's basic for a loop. I have run to store a mac address in an array.

Let's confirm our array. To confirm this, we will check this by printing our array output to serial monitor.

Make sure your loop counter is less than 10, or your esp might reboot due to overload. Our only goal is to check if data is stored or not.

    for(int i =0 ; 1<=10; ++i) {
        Serial.println(maclist[i][0]);
    }

I hope, it has also worked for you.


And it's completed a basic Mac sniffer, which will be used to sniff Mac. Yes, of course we will be continuing this and building a Mac sniffer for our office client, which will help us know that they are online or in our range.

So now our array, `maclist[64][3]` in which the 0 column is filled with mac addresses As discussed, we need to work with TTL (time to leave), which will be used to determine if the Mac is online or offline.

    void purge(){ // This maanges the TTL
        for(int i=0;i<=63;i++){
            if(!(maclist[i][0] == "")){
            int ttl = (maclist[i][1].toInt());
            ttl --;
        if(ttl <= 0){
        //Serial.println("OFFLINE: " + maclist[i][0]);
            maclist[i][2] = "OFFLINE";
            maclist[i][1] = defaultTTL;
        }else{
            maclist[i][1] = String(ttl);
      }
    }
This is not so confusing, the few things we have done. I will remind you, then you can easily understand them in one shot.
    if(!(maclist[i][0] == "")){
        int ttl = (maclist[i][1].toInt());
        ttl --;
Here I am checking if `maclist[i][0]` which contains mac, is not empty, then this `if` statify and proceeds further.
    
    int ttl = (maclist[i][1].toInt());
    ttl --;

So basically, whenever we get a packet that contains a mac address, we check if that mac address is already in the array that is received with the packet; if not, we need to store it in the array. but whenever we do anything, we are updating a 2nd column, which stores a TTL value. which we define as having a value of 60.
    ` maclist[i][1] = defaultTTL;`
And then we are reducing by 1  `ttl --` Now you will get to know why i used that logic to check if mac address is already present . let's take an example mac is stored in an array already . Now, with the help of the `purge()` function, it will start taking the ttl value, which is set to 60, and start reducing it by 1. and to be offline, it should be 0. But if the mac address is online, our logic will check if that mac address is already present or not(it will add if not ). If it is, it will just update the default TTL back to 60.

I hope this gets into your head. Let's move on to other things. We are only left with two functions. Bear with me, and we are done.

void updatetime(){ // This updates the time the device has been online for
  for(int i=0;i<=63;i++){
    if(!(maclist[i][0] == "")){
      if(maclist[i][2] == "")maclist[i][2] = "0";
      if(!(maclist[i][2] == "OFFLINE")){
          int timehere = (maclist[i][2].toInt());
          timehere ++;
          maclist[i][2] = String(timehere);
      }
      
      //Serial.println(maclist[i][0] + " : " + maclist[i][2]);
      
    }
  }
}

You can also skip this function. Specifically, if the user is offline, it will show that it is offline. If the user is online, it will show the timer how much he was online till.


This code is self-explanatory so here `if(maclist[i][2] == "")maclist[i][2] = "0";` i am checking if maclist array 2nd column is empty then it should be replace by `maclist[i][2] = "0";` by 0 string value . 
        int timehere = (maclist[i][2].toInt());
          timehere ++;
          maclist[i][2] = String(timehere);
So this is self-explanatory: as long as the 2nd column is not offline, then the  user is online. As discussed, whenever the user is online, we will be using a timer.



This checks if the MAC is in the reckonized list

    void showpeople(){ 
        for(int i=0;i<=63;i++){
            String tmp1 = maclist[i][0];
            if(!(tmp1 == "")){
                for(int j=0;j<=9;j++){
                    String tmp2 = ClientMac[j][1];
                    if(tmp1 == tmp2){
                        //Serial.print(ClientMac[j][0] + " : " + tmp1 + " : " + maclist[i][2] + "\n -- \n");
                    }
                }
            }
        }
    }
We need to build an array where we are going to store all our known people's details. We will be giving the name `ClientMac` as defined in the showpeople function.

    String ClientMac[10][2] = {
        {"Example Phone","121234567890"},
            
    };
All of us know what an array is, as we discussed in this article, so I have created an array of `ClientMac[10][2]` so you can increase the rows depending upon your need. How many employees do you need to monitor? if 20 then you can change it to `ClientMac[20][2]`

Now before finishing up we need to define some function to Set primary/secondary channel of ESP32.

#define maxCh 13 //max Channel -> US = 11, EU = 13, Japan = 14
int curChannel = 1;


Now that we are done with programming stuff, we just need to run this program.

void loop() {
    if(curChannel > maxCh){ 
      curChannel = 1;
    }
    esp_wifi_set_channel(curChannel, WIFI_SECOND_CHAN_NONE); //Set primary/secondary channel of ESP32.
    delay(1000);
    updatetime();
    purge();
    showpeople();
    curChannel++;
    
    
}

esp_wifi_set_channel:- Set primary/secondary channel of ESP32.
WIFI_SECOND_CHAN_NONE :- the channel width is HT20

Now We Are Done with Our Code Let's run this and check if it is compiling or not.