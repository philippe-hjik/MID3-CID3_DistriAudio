# MID3-CID3_DistriAudio

## Sommaire
- [Se connecter au broker MQTT avec le topic `test`](#Étape-1)
  
- [Détecter une demande de catalogue sur le topic et y répondre](#Étape-2)
  
- [Créer un bouton pour demander des catalogues avec un message MQTT](#Étape-3)
  
- [Création de classes selon le type de message](#Étape-4)

## Info
>[!IMPORTANT]
>Se référer à `Tiago`, `Thomas` ou `Philippe`  concernant [Création de classes selon le type de message](#Étape-4)
>Cette étape est toujours en phase de test aussi Les [Enumérateur selon les message](####-Enumération-des-types-de-messages)
## Étape 1
Se connecter au broker MQTT avec le topic `test`, configurer les variables d'authentification
```csharp


private IMqttClient mqttClient; // Client MQTT global
private MqttClientOptions mqttOptions; // Options de connexion globales                                 
private MqttClientFactory factory = new MqttClientFactory();

string broker = "mqtt.blue.section-inf.ch";  // Adresse du Broker
int port = 1883;
string clientId = Guid.NewGuid().ToString(); // création GUID

string topic = "";  // nom du topic
string username = "";
string password = "";


public async void createConnection()
{

        mqttClient = factory.CreateMqttClient();

        // Options de connexion MQTT
        mqttOptions = new MqttClientOptionsBuilder()
        .WithTcpServer(broker, port)
        .WithCredentials(username, password)
        .WithClientId(clientId)
        .WithCleanSession()
        .Build();

        // Se connecter au broker MQTT
        var connectResult = await mqttClient.ConnectAsync(mqttOptions);

        // Vérifier la connection au Broker
        if (connectResult.ResultCode == MqttClientConnectResultCode.Success)
        {
            MessageBox.Show("Connected to MQTT broker successfully.");

            // Se Subscribe with "No Local" option
            var subscribeOptions = new MqttClientSubscribeOptionsBuilder()
                .WithTopicFilter(f =>
                {
                    f.WithTopic(topic);
                    f.WithNoLocal(true); // Ensure the client does not receive its own messages
                })
                    .Build();
            // S'abonner à un topic
            await mqttClient.SubscribeAsync(subscribeOptions);
        }
}
```
## Étape 2
Détecter une demande de catalogue sur le topic et y répondre (toujours dans la même méthode pour l'instant)
```csharp

// Callback function when a message is received
mqttClient.ApplicationMessageReceivedAsync += async e =>
{
    string receivedMessage = Encoding.UTF8.GetString(e.ApplicationMessage.Payload);

    MessageBox.Show($"Received message: {receivedMessage}");

    // Vérifier que le message contient HELLO
    if (receivedMessage.Contains("HELLO") == true)
    {
        // Obtenir la liste des musiques à envoyer
        string musicList = GetMusicList();

        // Construisez le message à envoyer (sera changé en JSON)
        string response = $"{clientId} (Philippe) possède les musiques suivantes :\n{musicList}";
        
        if (mqttClient == null || !mqttClient.IsConnected)
        {
            MessageBox.Show("Client not connected. Reconnecting...");
            await mqttClient.ConnectAsync(mqttOptions);
        }

        // Créez le message à envoyer
        var message = new MqttApplicationMessageBuilder()
            .WithTopic(topic)
            .WithPayload(response)
            .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
            .WithRetainFlag(false)
            .Build();

        // Envoyez le message
        mqttClient.PublishAsync(message);
        Console.WriteLine("Message sent successfully!");
    }

    return;
};
```

## Étape 3
Créer un bouton pour demander des catalogues avec un message MQTT
- Envoi le message HELLO comme demande de catalogue
- Le fait de manière asynchrone
```csharp

private async void SendData(string data)
{
    // Créez le message à envoyer
    var message = new MqttApplicationMessageBuilder()
    .WithTopic(topic)
    .WithPayload(data)
    .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
    .WithRetainFlag(false)
    .Build();

    // Envoyez le message
     mqttClient.PublishAsync(message);
    Console.WriteLine("Message sent successfully!");
}
        
private async void button1_Click_1(object sender, EventArgs e)
{
    SendData("HELLO, qui a des musiques");
}
```

## Étape 4
Création de classes selon le type de message
- Demander des catalogues (Faire un broadcast)
- Envoyer son catalogue à tout le monde
- Transférer son catalogue à une personne sur son topic
>Un catalogue contient une `List<MediaData>` avec plusieurs musiques
> Dans la dernière version suggérée, Tiago fait une double Sérialisation de son contenu (La classe Enveloppe choisi et Le message MQTT en lui même) voir : [####-Classes-Générique](####-Classes-Générique)
#### Classes MediaData
```csharp
public class MediaData
{
    private string _file_name;
    private string _file_artist;
    private string _file_type;
    private long _file_size;
    private string _file_duration;

    public string File_name { get => _file_name; set => _file_name = value; }
    public string File_artist { get => _file_artist; set => _file_artist = value; }
    public string File_type { get => _file_type; set => _file_type = value; }
    public long File_size { get => _file_size; set => _file_size = value; }
    public string File_duration { get => _file_duration; set => _file_duration = value; }
    
}
```
#### Classes Générique
>[!TIP]
>La classe Générique permet de choisir entre les [classes de messages](####Enumération-des-types-de-messages)
>

#### Enumération des types de messages
>Voici les différents type de message possible
```csharp
public enum MessageType
{
    ENVOIE_CATALOGUE,
    DEMANDE_CATALOGUE,
    ENVOIE_FICHIER
}
```

#### Classes Générique
>La classe contenant la list de Media (Déja Sérialisé) selon le type de message et le Message MQTT
```csharp
public class GenericEnvelope
{
    string _senderId;
    MessageType _messageType;

    string _enveloppeJson;//classe specefique serialisee

    public MessageType MessageType { get => _messageType; set => _messageType = value; }
    public string SenderId { get => _senderId; set => _senderId = value; }
    public string EnveloppeJson { get => _enveloppeJson; set => _enveloppeJson = value; }
}
```

#### Classes Enveloppes
> Les classes enveloppe en question
```csharp
public class EnveloppeEnvoieCatalogue
{
    /* 
        type 1 ENVOIE_CATALOGUE
     */
    private int _type;
    private string _guid;
    private List<MediaData> _content;

    public string Guid { get => _guid; set => _guid = value; }
    public List<MediaData> Content { get => _content; set => _content = value; }
    public int Type { get => _type; set => _type = value; }
}

public class EnveloppeDemandeCatalogue
{
    /* 
        type 2 DEMANDE_CATALOGUE
     */
    private int _type;
    private string _guid;
    private string _content;

    public string Guid { get => _guid; set => _guid = value; }
    public string Content { get => _content; set => _content = value; }
    public int Type { get => _type; set => _type = value; }
}

public class EnveloppeEnvoieFichier
{
    /* 
        type 3 ENVOIE_FICHIER
     */
    private int _type;
    private string _guid;
    private string _content;

    public string Guid { get => _guid; set => _guid = value; }
    public string Content { get => _content; set => _content = value; }
    public int Type { get => _type; set => _type = value; }
}
```
### Exemple d'utilisation de l'énum
Ici lorsqu'on reçoit un message, on va regarder le type de l'enveloppe générique, pour trier

```csharp

private void ReiceiveMessage(MqttApplicationMessageReceivedEventArgs message)
        {
            try
            {
                Debug.Write(Encoding.UTF8.GetString(message.ApplicationMessage.Payload));
                GenericEnvelope enveloppe = JsonSerializer.Deserialize<GenericEnvelope>(Encoding.UTF8.GetString(message.ApplicationMessage.Payload));
                if (enveloppe.SenderId == clientId) return;
                switch (enveloppe.MessageType)
                {
                    case MessageType.ENVOIE_CATALOGUE:
                    {
                        EnvoieCatalogue enveloppeEnvoieCatalogue = JsonSerializer.Deserialize<EnvoieCatalogue>(enveloppe.EnveloppeJson);
                        break;
                    }
                    case MessageType.DEMANDE_CATALOGUE:
                    {
                        EnvoieCatalogue envoieCatalogue = new EnvoieCatalogue();
                        envoieCatalogue.Content = _maListMediaData;
                        SendMessage(mqttClient, MessageType.ENVOIE_CATALOGUE, clientId, envoieCatalogue, "test");
                        break;
                    }
                    case MessageType.ENVOIE_FICHIER:
                    {
                        EnvoieFichier enveloppeEnvoieFichier = JsonSerializer.Deserialize<EnvoieFichier>(enveloppe.EnveloppeJson);
                        break;
                    }
                }


            }
            catch (Exception ex)
            {
                Debug.WriteLine(ex.ToString());
            } 
        }
```
