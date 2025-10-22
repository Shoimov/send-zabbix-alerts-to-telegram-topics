# send-zabbix-alerts-to-telegram-topics
Send your Zabbix alerts to Telegram Supergroup Topics


### **This Project Includes Several Steps to Complete.**

### **Steps to do**
1.Create SuperGroup with Topics  
2.Create new bot with Botfather  
3. Configure Zabbix Media Types  
4. Change Zabbix User settings to use Media Type 


### **1.Create SuperGroup with Topics**

 Just Create simple group on Telegram and change its settings to add Topics. 

We need topics id to send messages right topics. In our example I have created several topics according to problem type:

<img width="2284" height="814" alt="Image" src="https://github.com/user-attachments/assets/5114a763-216e-4194-8923-c1afbfdfd3a1" />  


    

**to get topic and Group IDs we need to open  and login to  Web Telegram** ---> (https://web.telegram.org/)

Openyour super group and you will see chat id (Usually  like that :  -xxxxxxxxxxxx)  
<img width="1070" height="685" alt="Image" src="https://github.com/user-attachments/assets/a5a192f5-79bc-4071-9203-423c43d70134" />  

  
**to get topics Id open that topic and send some message. Copy that message link**
<img width="1774" height="694" alt="image" src="https://github.com/user-attachments/assets/ffbafa0f-f6f2-4cb3-8909-ec8bc0a2dd7e" />  

  
**it gives link like that**

https://t.me/c/3842340208/5/423  

 in that link  "5"  is your topic id. Get all topics IDs also likle that.  
 
**Write down all topics and chat id to one file and save it. We will need it later**

### 2.Create new bot with Botfather
This step is not hard. Just search botfather in telegram and create new bot. After that you should get bot token from botfather, save it.   Also add this bot to telegram group   

### 3. Configure Zabbix Media Types 

**Go to Alerts --> Media Types**
<img width="597" height="968" alt="image" src="https://github.com/user-attachments/assets/f6a14c95-d788-4c5d-8ee6-b4dbb20da822" />  

 **Create media type**  
 
 <img width="2560" height="1126" alt="image" src="https://github.com/user-attachments/assets/af3050db-1cf5-42f1-be54-6e14a3930e82" />  

   Add Parameters like in the picture.  
   TO  is your group ID (-326723676237236)  
   Token is your bot token  
   Also dont forget to enable this media  
   <img width="1864" height="981" alt="image" src="https://github.com/user-attachments/assets/0b0f03c4-a8b5-4295-82d8-8eb4d7b7d3b4" />



   **Also Add Following Code to the Script:**

```
var Telegram = {
    token: null,
    to: null,
    message: null,
    proxy: null,
    parse_mode: 'html',

    topics: {
        "Minor Problems": 19,
        "Disk IO": 20 ,
        "Critical": 21,
        "Cpu or Ram": 22,
        "Disk Full": 23,
        "Disk IO":24
    },

    escapeHTML: function (str) {
        if (!str) return "";
        return str
            .replace(/&/g, "&amp;")
            .replace(/</g, "&lt;")
            .replace(/>/g, "&gt;");
    },

    chooseTopic: function (text) {
        text = text.toLowerCase();
        if (text.includes("disk space is low") || text.includes("disk space is critically low")) return Telegram.topics["Disk Full"];
        if (text.includes("responses are too high")) return Telegram.topics["Disk IO"];
        if (text.includes("high memory utilization") || text.includes("load average is too high")) return Telegram.topics["Cpu or Ram"];
        if (text.includes("main humo channel") || text.includes("gci_ipsec") || text.includes("hypervisor is down") || text.includes("link down") || text.includes("unavailable by icmp ping"))
            return Telegram.topics["Critical"];
        return Telegram.topics["Minor Problems"];
    },

    sendMessage: function () {
        var topic_id = Telegram.chooseTopic(Telegram.message);
        var params = {
            chat_id: Telegram.to,
            text: Telegram.message,
            message_thread_id: topic_id,
            disable_web_page_preview: true,
            disable_notification: false,
            parse_mode: Telegram.parse_mode
        };

        var request = new HttpRequest();
        if (Telegram.proxy) request.setProxy(Telegram.proxy);
        request.addHeader('Content-Type: application/json');

        var url = 'https://api.telegram.org/bot' + Telegram.token + '/sendMessage';
        var data = JSON.stringify(params);

        Zabbix.log(4, '[Telegram Forward] URL: ' + url.replace(Telegram.token, '<TOKEN>'));
        Zabbix.log(4, '[Telegram Forward] params: ' + data);

        var response = request.post(url, data);
        Zabbix.log(4, '[Telegram Forward] HTTP code: ' + request.getStatus());

        try { response = JSON.parse(response); } catch (error) { response = null; }

        if (request.getStatus() !== 200 || !response || response.ok !== true) {
            var err = (response && response.description) ? response.description : 'Unknown error';
            throw 'Telegram send failed: ' + err;
        }
    }
};

try {
    var params = JSON.parse(value);

    if (!params.Token) throw 'Missing "Token" parameter';
    if (!params.To) throw 'Missing "To" parameter';

    Telegram.token = params.Token;
    Telegram.to = params.To;

    // --- Extract fields ---
    var msg = params.Message;
    var host = (msg.match(/Host:\s*(.*)/) || [])[1] || "Unknown host";
    var ip = (msg.match(/IP:\s*(.*)/) || [])[1] || ""; // <--- Extract IP if available
    var problem = (msg.match(/Problem name:\s*(.*)/) || [])[1] ||
                  (msg.match(/Problem:\s*(.*)/) || [])[1] ||
                  "Unknown problem";
    var data = (msg.match(/Operational data:\s*(.*)/) || [])[1] || "";
    var id = (msg.match(/Original problem ID:\s*(.*)/) || [])[1] || "";
 
    var subj = params.Subject.toLowerCase();
    var emoji = "⚪️";
    var titlePrefix = "";

    if (subj.includes("resolved") || subj.includes("recovery")) {
        emoji = "✅";
        titlePrefix = "Recovery: ";
    } else if (problem.toLowerCase().includes("disk")) {
        emoji = "💾";
    } else if (problem.toLowerCase().includes("cpu") || problem.toLowerCase().includes("memory")) {
        emoji = "🧠";
    } else if (problem.toLowerCase().includes("down") || problem.toLowerCase().includes("unavailable")) {
        emoji = "🚨";
    } else if (problem.toLowerCase().includes("warning")) {
        emoji = "🟡";
    } else if (problem.toLowerCase().includes("average")) {
        emoji = "🟠";
    } else if (problem.toLowerCase().includes("disaster")) {
        emoji = "🔥";
    }

    // --- Escape HTML safely ---
    host = Telegram.escapeHTML(host);
    ip = Telegram.escapeHTML(ip);
    problem = Telegram.escapeHTML(problem);
    data = Telegram.escapeHTML(data);
    id = Telegram.escapeHTML(id);

    // --- Combine host and IP ---
    var hostLine = ip ? host + " (" + ip + ")" : host;

    // --- Build message ---
    Telegram.message =
        '<b>' + emoji + ' ' + titlePrefix + problem + '</b>\n\n' +
        '🏷️ <b>Host:</b> ' + hostLine + '\n' +
        (data ? '💾 <b>Details:</b> ' + data + '\n' : '');

    Telegram.sendMessage();
    return 'OK';

} catch (error) {
    Zabbix.log(4, '[Telegram Forward] failed: ' + error);
    throw 'Sending failed: ' + error + '.';
}
```
You only need to change this part with your actual Topic ID and Topic Names:  


```
 topics: {
        "Minor Problems": 19,
        "Disk IO": 20 ,
        "Critical": 21,
        "Cpu or Ram": 22,
        "Disk Full": 23,
        "Disk IO":24
    },

    escapeHTML: function (str) {
        if (!str) return "";
        return str
            .replace(/&/g, "&amp;")
            .replace(/</g, "&lt;")
            .replace(/>/g, "&gt;");
    },
```
ALso change Topic names here:
```
    chooseTopic: function (text) {
        text = text.toLowerCase();
        if (text.includes("disk space is low") || text.includes("disk space is critically low")) return Telegram.topics["Disk Full"];
        if (text.includes("responses are too high")) return Telegram.topics["Disk IO"];
        if (text.includes("high memory utilization") || text.includes("load average is too high")) return Telegram.topics["Cpu or Ram"];
        if (text.includes("main humo channel") || text.includes("gci_ipsec") || text.includes("hypervisor is down") || text.includes("link down") || text.includes("unavailable by icmp ping"))
            return Telegram.topics["Critical"];
        return Telegram.topics["Minor Problems"];
    },
```

**Configuring Message Templates**  

  Open Message Templates tab and add following Templates:  
**1.**
    <img width="1464" height="439" alt="image" src="https://github.com/user-attachments/assets/ac87e994-fc71-47b2-b930-0daa19601b8c" />  


```
Problem started at {EVENT.TIME} on {EVENT.DATE}
Problem name: {EVENT.NAME}
Host: {HOSTNAME} ({HOST.IP})
Severity: {EVENT.SEVERITY}
Operational data: {EVENT.OPDATA}
Original problem ID: {EVENT.ID}
{TRIGGER.URL}
```
**2**  

<img width="736" height="369" alt="image" src="https://github.com/user-attachments/assets/b28d7b0e-d90e-40db-b058-ffbbe8742ff5" />  

```
Problem has been resolved in {EVENT.DURATION} at {EVENT.RECOVERY.TIME} on {EVENT.RECOVERY.DATE}
Problem name: {EVENT.NAME}
Host: {HOSTNAME} ({HOST.IP})
Severity: {EVENT.SEVERITY}
Original problem ID: {EVENT.ID}
{TRIGGER.URL}  
```
**3**  

<img width="736" height="369" alt="image" src="https://github.com/user-attachments/assets/c873ccd1-1dcf-4e63-b838-e03c4aa1425c" />  

```
{USER.FULLNAME} {EVENT.UPDATE.ACTION} problem at {EVENT.UPDATE.DATE} {EVENT.UPDATE.TIME}.
{EVENT.UPDATE.MESSAGE}

Current problem status is {EVENT.STATUS}, acknowledged: {EVENT.ACK.STATUS}.
```

 ### 4. Change Zabbix User settings to use Media Type  
 Go to Alerts --> Actions ---> Trigger Actions
 <img width="532" height="247" alt="image" src="https://github.com/user-attachments/assets/bf8b9c44-baad-4da2-b68f-b04c75c470b8" />  
 Create New Action  
 <img width="2323" height="421" alt="image" src="https://github.com/user-attachments/assets/43116bd1-7916-4f72-a507-1c5efefd19df" />  

   Go to Operations Tab and Add Following Configure to Operations, Recovery Operations and Update Operations Sections  
   <img width="1370" height="1004" alt="image" src="https://github.com/user-attachments/assets/a3c1259c-992d-41b9-a06c-ffd21f946852" />  
   <img width="772" height="601" alt="image" src="https://github.com/user-attachments/assets/76224bc2-d600-4594-8a5b-a0dc4a43e23f" />  

   Next Step is Go to Users --> Users and Select Admin  
   <img width="2554" height="525" alt="image" src="https://github.com/user-attachments/assets/6db6af3a-6af1-43d4-bb3b-11ec1eeb84f4" />  
   Select Media Tab and Add New Media as Telegram  
   <img width="2328" height="385" alt="image" src="https://github.com/user-attachments/assets/5d5f61e4-6578-4b7e-890d-b1bb0e8c5f6d" />



   _Send To_ should be same with your actual Telegram Group name  
   <img width="737" height="500" alt="image" src="https://github.com/user-attachments/assets/0a0f4dcd-0414-46e3-8ecf-4784be084830" />


  ### ** Troubleshooting**

  You have Just Configured  Sending Zabbix Alerts  to Telegram. For troubleshooting Go to Dashboard --> Current Problems and open Actions. It will tell you if there is problem with sending alerts  
  <img width="2555" height="770" alt="image" src="https://github.com/user-attachments/assets/bbb8ec47-face-4cfe-896c-f42e789eb049" />  

  Problem or success messages should be written here:  
  <img width="2247" height="693" alt="image" src="https://github.com/user-attachments/assets/bdd2b03a-df39-4ee5-b6ef-bb933213d617" />



















