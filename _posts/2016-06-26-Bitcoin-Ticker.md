---
layout: post
title: Hardware Bitcoin Price Ticker
---

So I happened upon the [ESP8266](https://en.wikipedia.org/wiki/ESP8266) microcontroller recently, a tiny low powered WiFi capable microchip intended for the internet of things (IOT). I've always been rather scepticle of the IOT movement and would rather not receive emails signed "yours truly, the toaster", but sometimes you have to just go with the flow.

Thus begun a project with some electronics, woodwork and programming, hours of fun...

The concept for the project was to produce an "clock" which rather than relaying the time would convey the current exchange rate of Bitcoin to USD. Following a little feature creep an android app to configure the price source and frequency of update along with upper and lower alarm bounds as well as additional currencies was added.


## The Electronic Hardware

* ESP8266 microcontroller module with serial to USB
* SSD1306 128*64 monochrome OLED display
* Passive piezo buzzer module with transistor driver

The microchip is supported by the [Arduino IDE](https://www.arduino.cc/en/Main/Software) via an [addon](https://github.com/esp8266/Arduino). The module includes breakouts of all the pins and an interface to USB to program the chip. Life never used to be this easy, in the past you could spend days just figuring out how to power, program and erase microchips.

![esp8266](/img/esp8266.jpg)

## Straight to Coding

The coding went fast as libraries were available for the screen and the Arduino IDE abstracts much of what would once have had to be gleaned from datasheets. The basic elements of the code are as follows:

* Setup serial connection for debug messages.
{% highlight c %}
Serial.begin(115200);
delay(10);
  Serial.println(F("Bitcoin Price Ticker"));
{% endhighlight %} 
* Use the [WifiManager](https://github.com/tzapu/WiFiManager) library to either connect to the internet or start an access point with captive portal to allow the user to configure SSID/password.
{% highlight c %}
  WiFiManager wifiManager;
  wifiManager.autoConnect("Bitcoin-Ticker");  

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.print(F("IP address: "));
  Serial.println(WiFi.localIP()); 
{% endhighlight %}
* Get the price from an online price feed by connecting to the remote server
	{% highlight c %}
  if (!client->connect(host, port)) {
    Serial.println(F("ERROR conn failed"));
    delay(1);
    return false;
  }
  
  Serial.print(F("Requesting URL: "));
  Serial.println(url);
  delay(1);
  // send request to server
  client->print(String("GET ") 
  + url 
  + " HTTP/1.1\r\n" 
  + "Host: " 
  + host 
  + "\r\n" 
  + "Connection: close\r\n\r\n");
 {% endhighlight %}
and storing the response for further processing.
{% highlight c %}
 void read_content(WiFiClient* client, char* content, size_t maxsize){
   size_t length = client->readBytes(content, maxsize);
   content[length] = '\0';
   Serial.println(content); 
 }
{% endhighlight %}

* Discard everything returned by the server bar the JSON data that we need. As JSON data is delivered in curly braces "{}", we only have to parse the servers response until the opening brace is found.
{% highlight c %}
  char brace = '{';
  bool check = client->find("{");

  return check;
{% endhighlight %}
* Use the [ArdiunoJSON Library](https://github.com/bblanchon/ArduinoJson) to parse the result and extract the price.
{% highlight c %}
  StaticJsonBuffer<512> jsonBuffer;

  JsonObject& root = jsonBuffer.parseObject(response);

  if (!root.success()){
    Serial.println(F("JSON failed!"));
    return false;
  }
  Serial.print(F("JSON = "));
  Serial.println(jsonBuffer.size());
  char result[8];
  switch (currency){
    case 0://cd usd
        strncpy(result, root["bpi"]["USD"]["rate"], 7);
        break;
    case 1://cd eur
       strncpy(result, root["bpi"]["EUR"]["rate"], 7);     
       break;
    case 2://cd gbp
       strncpy(result, root["bpi"]["GBP"]["rate"], 7);
       break;
    case 3://houbi
       strncpy(result, root["ticker"]["last"], 7);
       break;
    case 4://okcoin
       strncpy(result, root["ticker"]["last"], 7);
       break;
    case 5://bitfinexBTCETHLTC
       strncpy(result, root["mid"], 7);
       break;
    default:
       return false;
  }
  result[7] = '\0';
{% endhighlight %}
* Display the price!
{% highlight c %}
 display.setCursor(xoffset, yoffset);
 display.print(value);
 display.display();
{% endhighlight %}
 
The full code is available on my [github repository](https://github.com/rabfulton/HardwareBitcoinTicker). Please feel free to point out momentously stupid errors at will.
 
## Making an Enclosure
 
Now to the hard part.
 
I spent a good few days mulling over how to make a nice case for this project. Not everyone has a 3D printer and CNC lying around.

To create a curved fascia for the screen I decided to try laminating some Mahogany veneer I had lying around. To do this one creates a mould and applies glue to multiple layers of veneer before clamping them in the form.

 ![Bitcoin Ticker Case](/img/btc1.jpg)

The veneers are only 0.6mm thick and bend easily without breaking with the grain orientated perpendicular to the curve. A good amount of dry soap can be rubbed on the form to prevent any excess glue sticking to it.

 ![Bitcoin Ticker Case](/img/btc2.jpg)

Once the glue is dried the veneers will retain the shape of the form. This is basically a home made plywood of the mahogany variety (not available at your local builders merchant).

 ![Bitcoin Ticker Case](/img/btc3.jpg)

The rosewood sides are backed by some ply and glued onto the the fascia.

 ![Bitcoin Ticker Case](/img/btc4.jpg)

A wooden block in the vice behind the fascia allows the opening for the screen to be chiselled out with just a few careful blows from a mallet.
 
 ![Bitcoin Ticker Case](/img/btc5.jpg)

The rosewood used here is a am old saw cut veneer about 1.5mm thick and is brittle. Accordingly it has been glued onto a maple backing veneer to make it easier to work with. In the picture is also a ply template used to mark and outline of the faceplate.

 ![Bitcoin Ticker Case](/img/btc6.jpg)
  
The face plate glued in position.

 ![Bitcoin Ticker Case](/img/btc7.jpg)

I chose to use brass rod to attach the screens mounting holes to the case. Getting the alignment correct is somewhat difficult. I start with one corner drilled and then run a short program to draw a rectangle and two diagonal lines from corner to corner. The screen can then be adjusted and the holes for drilling marked.

{% highlight c %}
  display.drawRect(0,0,128,64,WHITE);
  display.drawLine(0,0,127,63,WHITE);
  display.drawLine(0,63,127,0,WHITE);
  display.display();
{% endhighlight %}

