# MID3-CID3_DistriAudio

## Sommaire
- [Se connecter au broker MQTT avec le topic `test`](#Étape-1)
- [Déteter une demande de catalogue sur le topic et y répondre](#Étape-2)
- [Créer un bouton pour demander des catalogues avec un message MQTT](#Étape-3)

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


public async void creatConnection()
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
Déteter une demande de catalogue sur le topic et y répondre (toujours dans la même méthode pour l'instant)
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

## Étape 2
Créer un bouton pour demander des catalogues avec un message MQTT
- Envoi le message HELLO comme demande de catalogue
- Le fait de manière asynchrone
```csharp

private async void SendData(string data)
{
    // Create a MQTT client instance
    var mqttClient = factory.CreateMqttClient();

    // Create MQTT client options
    var options = new MqttClientOptionsBuilder()
        .WithTcpServer(broker, port) // MQTT broker address and port
        .WithCredentials(username, password) // Set username and password
        .WithClientId(clientId)
        .WithCleanSession()
        .Build();

    // Connectez-vous au broker MQTT
    var connectResult = await mqttClient.ConnectAsync(options);

    var message = new MqttApplicationMessageBuilder()
        .WithTopic(topic)
        .WithPayload(data)
        .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
        .WithRetainFlag()
        .Build();

    await mqttClient.PublishAsync(message);
    await Task.Delay(1000); // Wait for 1 second

    mqttClient.UnsubscribeAsync(topic);
    mqttClient.DisconnectAsync();

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

#### Enumération des types de messages
```csharp
public enum MessageType
{
    ENVOIE_CATALOGUE,
    ENVOIE_FICHIER,
    DEMANDE_CATALOGUE
}
```

#### Enumération des types de messages
```csharp

```


