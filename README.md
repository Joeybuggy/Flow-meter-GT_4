const char FW_VERSION[8] = "D_1.2.0";  //FW Version
const char pwd_id[13] = "94103ED40C74";// "94103ED40E04 Lorenzo"; //94103ED40760 sean //TODO: Read this from cloud and store in EEPROM




volatile double flow_frequency_0; //measuring the rising edges of the signal
volatile double flow_frequency_1; 
volatile double flow_frequency_2; 
volatile double flow_frequency_3; 
volatile double flow_frequency_4; 


long Calc_0; //total of the pulse count in one event.                               
long Calc_1;
long Calc_2;                               
long Calc_3;
long Calc_4;


volatile unsigned long last_micros;
long debouncing_time = 15;

int hallsensor_0 = D4;    // the 1st flow meter connect to pin D4
int hallsensor_1 = D1;    // the 2nd to pin D1
int hallsensor_2 = D2;    // the 3rd to pin D2
int hallsensor_3 = D3;    // the 4th to pin D3
int hallsensor_4 = D5;    // the 5th to pin D5

const int PIN_LED = D7; // This is the LED that is already on your device. This is for debug.


//mark the status of the flow meter
#define UNKNOWN -1
#define NOT_CALIBRATED 0
#define CALIBRATED 1
#define CLOSED_ONE 2
#define OPENED_ONE 3
#define OPENED_TWO 4
#define CLOSED_TWO 5
volatile byte lastStatus_0 = UNKNOWN;    //Last Status for the 1st flow meter
volatile byte lastStatus_1 = UNKNOWN;    //Last Status for the 2nd flow meter
volatile byte lastStatus_2 = UNKNOWN;    //Last Status for the 3rd flow meter
volatile byte lastStatus_3 = UNKNOWN;    //Last Status for the 4th flow meter
volatile byte lastStatus_4 = UNKNOWN;    //Last Status for the 5th flow meter



int status_ts;   // store the timestamp
char buffer_open[10];
String strResult;

void rpm_0 ()     //This is the function that the interupt calls 
{ 
    if((micros() - last_micros ) >= debouncing_time * 1000)   //debounce
    {
        flow_frequency_0++;  
        last_micros = micros();
    }
//hall effect sensors signal
} 

void rpm_1 ()  
{ 
    if((micros() - last_micros) >= debouncing_time * 1000)
    {
        flow_frequency_1++;  
        last_micros = micros();
    }
} 
void rpm_2 ()  
{ 
    if((micros() - last_micros) >= debouncing_time * 1000)
    {
        flow_frequency_2++;  
        last_micros = micros();
    }
} 
void rpm_3 ()     
{ 
    if((micros() - last_micros) >= debouncing_time * 1000)
    {
        flow_frequency_3++;  
        last_micros = micros();
    }
} 

void rpm_4 ()    
{ 
    if((micros() - last_micros) >= debouncing_time * 1000)
    {
        flow_frequency_4++; 
        last_micros = micros();
    }
} 

// flashLED for debug
void flashLed(int period, int T)
{
    for(int i = 0; i < T; i++)
    {
        digitalWrite(PIN_LED, HIGH);    
        delay(period);
        digitalWrite(PIN_LED, LOW);
        delay(period);
    }
}

void setup() //
{ 
 Serial.begin(9600); //This is the setup function where the serial port is 
 pinMode(PIN_LED, OUTPUT);
 pinMode(hallsensor_0, INPUT_PULLUP); //initializes digital pin as an input
 pinMode(hallsensor_1, INPUT_PULLUP); 
 pinMode(hallsensor_2, INPUT_PULLUP); 
 pinMode(hallsensor_3, INPUT_PULLUP); 
 pinMode(hallsensor_4, INPUT_PULLUP); 


//initialised,
 attachInterrupt(hallsensor_0, rpm_0, RISING); //and the interrupt is attached
 attachInterrupt(hallsensor_1, rpm_1, RISING); 
 attachInterrupt(hallsensor_2, rpm_2, RISING); 
 attachInterrupt(hallsensor_3, rpm_3, RISING); 
 attachInterrupt(hallsensor_4, rpm_4, RISING); 
} 
// the loop() method runs over and over again,
// as long as the Arduino has power
void loop ()    
{
 flow_frequency_0 = 0;      //Set NbTops to 0 ready for calculations
 flow_frequency_1 = 0;      
 flow_frequency_2 = 0;      
 flow_frequency_3 = 0;       
 flow_frequency_4 = 0;       
 
 interrupts();           //Enables interrupts
 delay (300);      // It count the total pulus between enable and disable
 noInterrupts();            //Disable interrupts

 
 if(flow_frequency_0 > 1 ) //if there is flow/pules. Use "> 1" to debounce the vibration when there is a fixture is being used close by this fixture. 
                           //Ex: when the tub is filling water, the vibration makes the flow meter that is on the shower head read false positive.
 {
    if(lastStatus_0 != OPENED_ONE)   // if the status change, do the following 
    {
        delay(100);
        flow_frequency_0 = 0;      
        interrupts();           
        delay (500);      // count pulses when the interrrupts is enable
        noInterrupts();            

        if(flow_frequency_0 > 1)   // check the pulse again to debounce false positive. 
        {
            status_ts = Time.now(); // record the time when there is water running
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'o','t':'sc','ts':%s}]}"  
                            , pwd_id, buffer_open);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            flashLed(50, 4);
            lastStatus_0 =  OPENED_ONE; // change status after upload the open event 
            Calc_0 = flow_frequency_0;  // count pulses 
        }
        
    }   
    else {Calc_0 += flow_frequency_0;} // add up the pulses 
 }
 else // if there is no flow/pules
 {
    if(lastStatus_0 != CLOSED_ONE) // status has changed
    {
        
        delay(100);
        flow_frequency_0 = 0;      
        interrupts();       
        delay (500);      //count pulses when the interrrupts is enable
        noInterrupts();            
        
        if(flow_frequency_0 == 0) //if there no flow/pules
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'c','t':'sc','ts':%s,'f':%d}]}"  
                            , pwd_id, buffer_open, Calc_0);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            flashLed(50, 4);
            lastStatus_0 =  CLOSED_ONE;  //chhange status after uplad the close event
            Calc_0 = 0;
        }
        
    }
 }

 if(flow_frequency_1 > 1 )
 {
    if(lastStatus_1 != OPENED_TWO)
    {
        delay(100);
        flow_frequency_1 = 0;      
        interrupts();           
        delay (500);      
        noInterrupts();            

        if(flow_frequency_1 > 1)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'o','t':'sh','ts':%s}]}"  
                            , pwd_id, buffer_open);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            flashLed(30, 1);
            lastStatus_1 =  OPENED_TWO;
            Calc_1 = flow_frequency_1;
        }
    }
    else {Calc_1 += flow_frequency_1;}
 }
 else
 {
    if(lastStatus_1 != CLOSED_TWO) 
    {
        delay(100);
        flow_frequency_1 = 0;      
        interrupts();           
        delay (500);     
        noInterrupts();  

        if(flow_frequency_1 == 0)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'c','t':'sh','ts':%s,'f':%d}]}"  
                            , pwd_id, buffer_open, Calc_1);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            Serial.println("1 is closed");
            flashLed(60, 1);
            lastStatus_1 =  CLOSED_TWO;
            Calc_1 = 0;
        }
    }
 } 


 if(flow_frequency_2 > 1 )
 {
    if(lastStatus_2 != OPENED_TWO)
    {
        delay(100);
        flow_frequency_2 = 0;      
        interrupts();           
        delay (300);      
        noInterrupts();            

        if(flow_frequency_2 > 1)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'o','t':'c','ts':%s}]}"
                            , pwd_id, buffer_open);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            Serial.println("2 is opened");
            flashLed(30, 2);
            lastStatus_2 =  OPENED_TWO;
            Calc_2 = flow_frequency_2;
        }
    }    
    else {Calc_2 += flow_frequency_2;}
 }
 else
 {
    if(lastStatus_2 != CLOSED_TWO) 
    {
        delay(100);
        flow_frequency_2 = 0;      
        interrupts();           
        delay (1000);      
        noInterrupts();    

        if(flow_frequency_2 == 0)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'c','t':'c','ts':%s,'f':%d}]}"  
                            , pwd_id, buffer_open, Calc_2);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            Serial.println("2 is closed");
            flashLed(20, 2);
            lastStatus_2 =  CLOSED_TWO;
            Calc_2 = 0;
        }
    }
 } 
 
 if(flow_frequency_3 > 1 )
 {
    if(lastStatus_3 != OPENED_TWO)
    {
        delay(100);
        flow_frequency_3 = 0;      
        interrupts();           
        delay (300);     
        noInterrupts();  

        if(flow_frequency_3 > 1)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'o','t':'h','ts':%s}]}" 
                            , pwd_id, buffer_open);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            Serial.println("3 is opened");
            flashLed(50, 3);
            lastStatus_3 =  OPENED_TWO;
            Calc_3 = flow_frequency_3;
        }
    }    
    else {Calc_3 += flow_frequency_3;}
 }
 else
 {
    if(lastStatus_3 != CLOSED_TWO) 
    {
        delay(100);
        flow_frequency_3 = 0;      
        interrupts();           
        delay (1000);      
        noInterrupts();    

        if(flow_frequency_3 == 0)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'c','t':'h','ts':%s,'f':%d}]}"  
                            , pwd_id, buffer_open, Calc_3);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            Serial.println("3 is closed");
            flashLed(20, 3);
            lastStatus_3 =  CLOSED_TWO;
            Calc_3 = 0;
        }
    }
 } 
 
  if(flow_frequency_4 > 1 )
 {
    if(lastStatus_4 != OPENED_TWO)
    {
        delay(100);
        flow_frequency_4 = 0;      
        interrupts();       
        delay (500);      
        noInterrupts();    

        if(flow_frequency_4 > 1)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'o','t':'t','ts':%s}]}" 
                            , pwd_id, buffer_open);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            flashLed(50, 4);
            lastStatus_4 =  OPENED_TWO;
            Calc_4 = flow_frequency_4;
        }
    }    
    else {Calc_4 += flow_frequency_4;}
 }
 else
 {
    if(lastStatus_4 != CLOSED_TWO) 
    {
        delay(100);
        flow_frequency_4 = 0;      
        interrupts();           
        delay (300);      
        noInterrupts();   

        if(flow_frequency_4 == 0)
        {
            status_ts = Time.now();
            itoa(status_ts, buffer_open, 10);
            strResult = String::format("{'home_id':'%s','events': [{'e':'c','t':'t','ts':%s,'f':%d}]}"  
                            , pwd_id, buffer_open, Calc_4);
            Spark.publish("sendgtdata2", strResult, 60, PRIVATE);
            flashLed(50, 4);
            lastStatus_4 =  CLOSED_TWO;
            Calc_4 = 0;
        }
    }
 } 

}
