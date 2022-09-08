---
slug: 06-functions-dotnet
title: 06. Functions for .NET Devs
authors: [mike]
draft: true
hide_table_of_contents: false 
toc_min_heading_level: 2
toc_max_heading_level: 3
keywords: [azure, functions, serverless, concepts]
image: ./img/banner.png
description: "Introduction to Azure Functions, from core concepts to hello world!" 
tags: [serverless-september, 30-days-of-serverless, serverless-hacks, zero-to-hero, ask-the-expert, azure-functions, azure-container-apps, azure-event-grid, azure-logic-apps, serverless-e2e]
---

<head>
  <meta name="twitter:url" 
    content="https://azure.github.io/Cloud-Native/blog/functions-1" />
  <meta name="twitter:title" 
    content="#30DaysOfServerless: Azure Functions Fundamentals" />
  <meta name="twitter:description" 
    content="#30DaysOfServerless: Azure Functions Fundamentals" />
  <meta name="twitter:image"
    content="https://azure.github.io/Cloud-Native/img/banners/post-kickoff.png" />
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:creator" 
    content="@nitya" />
  <meta name="twitter:site" content="@AzureAdvocates" /> 
  <link rel="canonical" 
    href="https://azure.github.io/Cloud-Native/blog/06-functions-dotnet" />
</head>

---

Welcome to `Day 6` of #30DaysOfServerless!

The theme for this week is Azure Functions. We'll talk about ...

---

## What We'll Cover
 * Azure Functions and Serverless explained.
 * Exercise: Provision a new Functions App.
 * Exercise: Creating a Function with a HTTP trigger.
 * Resources: For self-study!

![](./img/banner.png)

---


## Azure Functions and Serverless explained.
Azure Functions is a serverless compute service that enables you to run code on-demand without explicitly provisioning or managing the underlying infrastructure. 

Developing code that will run in a serverless environment departs from the traditional ASP.NET WebAPI technique of creating controllers with actions. Instead, as the name implies, we make functions as small snippets of functionality. An Azure Function persists these code snippets and meta-information concerning when and how it should get executed. The function sleeps until it has been invoked by a trigger, which wakes it up, runs the code snippet and returns to sleep.

This behaviour allows for a very attractive pricing model where you only pay for the execution time of an Azure Function. If you write code that never gets executed, it won't cost you anything! That means you only pay and Azure Function when it is actually used.

# Using Azure Functions to solve a real problem 
 
Let's look at how I'm using Azure Functions to help me learn Italian and overcome the difficulty of learning pronunciation!

The idea is simple. I want to create an app that enables me to translate a word or phrase from English and hear its pronunciation at normal speed and half-speed. Since I'll be the only user and might go days or weeks between creating new phrases, Azure Functions provides the perfect hosting environment for this project! 

To make this possible, I'll utilise three additional Azure Services, [Azure Storage](https://docs.microsoft.com/en-us/azure/storage/common/storage-introduction), [Cognitive Services Translator](https://docs.microsoft.com/en-us/azure/cognitive-services/translator/) and [Text-to-Speech](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/index-text-to-speech). I'll develop it in [Visual Studio Code](https://code.visualstudio.com/) on my PC, as it has [excellent support](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=csharp) for developer Azure Functions and works across all the platforms. 

As mentioned previously, Azure Functions uses the concept of [triggers and bindings](https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings?tabs=csharp), which lets the function interact with various Azure services with minimal code required. In this case, the app will use an [HTTP trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook?tabs=in-process%2Cfunctionsv2&pivots=programming-language-csharp), which means it'll run when receiving an HTTP request. During it's execution, it creates the audio files and then saves to Azure Storage using the [Blob storage output binding](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob-output?tabs=in-process%2Cextensionv5&pivots=programming-language-csharp).

For the sake of beverity, let use the following the guide from the [docs](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-csharp?tabs=in-process) to create a new function in C# with the HTTP trigger before we dig into how to create the translator function.

Once we've followed the guide, we should have a function that executes upon receiving a HTTP request. The entry point is the Run method, which only contains the trigger and a logger and probably looks a little like:

```csharp
[FunctionName("TranslateToItalian")]
public static async Task<HttpResponseMessage> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
    ILogger log)
{
  // Implementation goes here.
}
```

## Run Arguments

As we know we're going to be creating two audio files and saving them to Azure Storage, lets add the output bindings now. 

```csharp
public static async Task<HttpResponseMessage> Run( 
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req, 

            //Output Bindings
            [Blob("translations/{rand-guid}.mp3", FileAccess.ReadWrite, Connection = "AzureWebJobsStorage")] BlockBlobClient halfSpeedAudioBlob,
            [Blob("translations/{rand-guid}.mp3", FileAccess.ReadWrite, Connection = "AzureWebJobsStorage")] BlockBlobClient normalSpeedAudioBlob,
            ILogger log)
        {
```
Lets explore the ```Blob``` attribute in detail. 

Notice the ```{rand-guid}``` in the path to the blob. This is a good example of [binding expressions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-expressions-patterns), which are the most powerful feature of triggers and bindings! In this case, we'll use the [Create GUIDs](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-expressions-patterns#create-guids) binding expression to ensure the audio file names are unique. 

Next up is the file access property, which is self explanitory, followed by the connection property. 
The Connection property allows us to point to an alternative storage account should we want to, but in this case we'll keep it using the default account assoicated with the Function.  

After the ```Blob``` attribute, we find the [BlockBlobClient](https://docs.microsoft.com/en-us/javascript/api/@azure/storage-blob/blockblobclient?view=azure-node-latest) argument, which provides a blob object that we can use to save the audio data to. 


## Translation 

We'll keep a similiar implemention for capturing the english phrase as the current HTTP trigger implementation. 

```csharp
string englishInput = req.Query["input"];
string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
dynamic data = JsonConvert.DeserializeObject(requestBody);
englishInput = englishInput ?? data?.englishInput;

// Check the englishInput for null and handle appropriately 
```

Let's implement the translation as a seperate method named ```TranslateText``` and we'll use the [Cognitive Services Translator REST API](https://docs.microsoft.com/en-us/azure/cognitive-services/translator/reference/v3-0-reference).

```csharp

  // The Translation object
  public class Translation
  {
      [JsonProperty("text")]
      public string Text { get; set; }

      [JsonProperty("to")]
      public string To { get; set; }
  }

  // The translation implementation 
  private static async Task<string> TranslateText(string englishInput, ILogger log)
  {
      var route = "/translate?api-version=3.0&from=en&to=it";
      var body = new object[] { new { Text = englishInput } };
      var requestBody = JsonConvert.SerializeObject(body);

      using (var client = new HttpClient())
      using (var request = new HttpRequestMessage())
      {
          // Build the request.
          request.Method = HttpMethod.Post;
          request.RequestUri = new Uri(_translatorEndpoint + route);
          request.Content = new StringContent(requestBody, Encoding.UTF8, "application/json");
          request.Headers.Add("Ocp-Apim-Subscription-Key", _translatorApiKey);
          request.Headers.Add("Ocp-Apim-Subscription-Region", _region);      

          // Send the request and get response.
          var response = await client.SendAsync(request).ConfigureAwait(false);
          // Read and return response as a string.
          var json = await response.Content.ReadAsStringAsync();       
          JArray a = JArray.Parse(json);
          string translatedText = string.Empty;
          foreach (JObject o in a.Children<JObject>())
          {
              foreach (JProperty p in o.Properties())
              {
                  foreach (JObject s in p.Value.Children<JObject>())
                  {
                      var translation = JsonConvert.DeserializeObject<Translation>(s.ToString());
                      translatedText = translation.Text;
                  }
              }
          }
          log.LogInformation($"Translated: {englishInput} -> {translatedText}");
          return translatedText;
      }
  }
```

The method contains a few variables that are defined within the class. To add these, you'll want to add the following to the class: 

```csharp
private static string _translatorApiKey => Environment.GetEnvironmentVariable("TranslatorKey");
private static string _translatorEndpoint => Environment.GetEnvironmentVariable("TranslatorEndpoint");
private static string _region => Environment.GetEnvironmentVariable("TranslationRegion");
```

When developing and testing locally, you can use the [local settings](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v4%2Cmacos%2Ccsharp%2Cportal%2Cbash#local-settings) file named ```local.settings.json```, but you'll need to remember to set [create and set](https://docs.microsoft.com/en-us/azure/azure-functions/functions-app-settings) these variables in the Functions app settings when deploying to Azure! 

We can now invoke the ```TranslateText``` method from the Run method. 

```csharp
// Translate the input to Italian.
var translatedText = await TranslateText(englishInput, log);
```

## Text to Speech

The final piece of the puzzle is to add the text to speech functionality using [Cognitive Services Text to Speech](https://azure.microsoft.com/en-us/services/cognitive-services/text-to-speech/#overview). Thankfully the team have created an [SDK](https://www.nuget.org/packages/Microsoft.CognitiveServices.Speech) and great [documentation](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/get-started-speech-to-text) making the service incredibly easy to use! To add the SDK, let's use the .NET CLI: 

```
dotnet add package Microsoft.CognitiveServices.Speech --version 1.23.0
```

Let's define another variable under the ```_region``` variable which we'll name ```_textToSpeechApiKey```. Make sure to also add the key and value to your local settings. 

```csharp
private static string _textToSpeechApiKey => Environment.GetEnvironmentVariable("TextToSpeechKey");
```

The implementation of the text to speech functionality is mostly provided by the quick start docs.

```csharp
private static async Task ConvertTextToSpeechAndSave2(string text, BlockBlobClient blob, ILogger log)
{
    log.LogInformation("Starting ConvertTextToSpeechAndSave");

    var speechConfig = SpeechConfig.FromSubscription(_textToSpeechApiKey, _region);
    speechConfig.SetSpeechSynthesisOutputFormat(SpeechSynthesisOutputFormat.Audio16Khz32KBitRateMonoMp3);
    // The language of the voice that speaks.
    speechConfig.SpeechSynthesisVoiceName = "it-IT-LisandroNeural";

    // We save a copy of the translation locally (these are wiped with each new deployment of the function).
    var tmpDir = Directory.CreateDirectory(Path.Combine(Path.GetTempPath(), "translations"));
    var filePath = Path.Combine(tmpDir.FullName, blob.Name);

    using(var fileOutput = AudioConfig.FromWavFileOutput(filePath))
    using(var speechSynthesizer = new SpeechSynthesizer(speechConfig, fileOutput))
    {
        var speechSynthesisResult = await speechSynthesizer.SpeakTextAsync(text);
        switch(speechSynthesisResult.Reason)
        {
            case ResultReason.SynthesizingAudioCompleted:          
                // Upload to Blob Storage.
                log.LogInformation("Uploading audio to blob storage");

                // Store the audio file into Blob storage. It's SUPER easy with output bindings!!
                await blob.UploadAsync(new MemoryStream(speechSynthesisResult.AudioData), new Azure.Storage.Blobs.Models.BlobUploadOptions());
                log.LogInformation($"Completed uploading audio to blob storage: {blob.Uri}");
                return;

            case ResultReason.Canceled:
                var cancellation = SpeechSynthesisCancellationDetails.FromResult(speechSynthesisResult);
                log.LogError($"Translation Cancelled. {cancellation.Reason}");
                return;
        }
    }  
}
```

### SSML

While the above code should work, it doesn't allow for setting the speech speed. To enable this, we need to use the [Speech Synthesis Markup Language](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/speech-synthesis-markup?tabs=csharp)(SSML) and tweak the our implementation. 

Lets first create an XML file named ```SlowedTTS.xml``` and ensure it's copied to the output directory on build. It's content can be copied from the Text to Speech demo page. 

```xml
<speak xmlns="http://www.w3.org/2001/10/synthesis" xmlns:mstts="http://www.w3.org/2001/mstts" xmlns:emo="http://www.w3.org/2009/10/emotionml" version="1.0" xml:lang="en-US">
	<voice name="it-IT-LisandroNeural">
		<prosody rate="-45%" pitch="0%">
			<speak xmlns="http://www.w3.org/2001/10/synthesis" xmlns:mstts="http://www.w3.org/2001/mstts" xmlns:emo="http://www.w3.org/2009/10/emotionml" version="1.0" xml:lang="en-US">
				<voice name="it-IT-LisandroNeural">
					<prosody rate="-45%" pitch="0%">Ciao</prosody>
				</voice>
			</speak>
		</prosody>
	</voice>
</speak>
```

We can then add a few new classes which we'll use to deserialise the XML for editing. 

```csharp
 // Text to Speech SSML request 
[XmlRoot(ElementName = "prosody", Namespace = "http://www.w3.org/2001/10/synthesis")]
public class Prosody
{
    [XmlAttribute(AttributeName = "rate")]
    public string Rate { get; set; }

    [XmlAttribute(AttributeName = "pitch")]
    public string Pitch { get; set; }

    [XmlText]
    public string Text { get; set; }
}

[XmlRoot(ElementName = "voice", Namespace = "http://www.w3.org/2001/10/synthesis")]
public class Voice
{
    [XmlElement(ElementName = "prosody", Namespace = "http://www.w3.org/2001/10/synthesis")]
    public Prosody Prosody { get; set; }

    [XmlAttribute(AttributeName = "name")]
    public string Name { get; set; }
}

[XmlRoot(ElementName = "speak", Namespace = "http://www.w3.org/2001/10/synthesis")]
public class Speak
{
    [XmlElement(ElementName = "voice", Namespace = "http://www.w3.org/2001/10/synthesis")]
    public Voice Voice { get; set; }

    [XmlAttribute(AttributeName = "xmlns")]
    public string Xmlns { get; set; }

    [XmlAttribute(AttributeName = "mstts", Namespace = "http://www.w3.org/2000/xmlns/")]
    public string Mstts { get; set; }

    [XmlAttribute(AttributeName = "emo", Namespace = "http://www.w3.org/2000/xmlns/")]
    public string Emo { get; set; }

    [XmlAttribute(AttributeName = "version")]
    public string Version { get; set; }

    [XmlAttribute(AttributeName = "lang", Namespace = "http://www.w3.org/XML/1998/namespace")]
    public string Lang { get; set; }
}
```

### Loading the XML 

To read the SSML XML file can use the ```ExecutionContext``` object to resolve the local path of the function. To do this, lets update our Functions run method. 

```csharp
public static async Task<HttpResponseMessage> Run( 
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req, 

            //Output Bindings
            [Blob("translations/{rand-guid}.mp3", FileAccess.ReadWrite, Connection = "AzureWebJobsStorage")] BlockBlobClient halfSpeedAudioBlob,
            [Blob("translations/{rand-guid}.mp3", FileAccess.ReadWrite, Connection = "AzureWebJobsStorage")] BlockBlobClient normalSpeedAudioBlob,
            ExecutionContext context,
            ILogger log)
        {
```

Now we'll modify the arguments of ```ConvertTextToSpeechAndSave``` by adding the ```ExecutionContext``` and a ```float``` to represent the requested audio speed/rate.

```csharp
//The update method signature
private static async Task ConvertTextToSpeechAndSave(string text, float rate, BlockBlobClient blob, ExecutionContext context, ILogger log)
```

To create the XML string needed for the SSML API, we'll create a method for reading the XML file, deserialising it, setting the values and converting it back to XML. While it's not the most efficient, it should make it easier to tweet other values in the future if we want. 

```csharp
private static string TextToSpeechSettingsXml(ExecutionContext context, string text, float rate)
{
    var invertedRate = -(1.0f - rate);
    var filePath = Path.GetFullPath(Path.Combine(context.FunctionDirectory, "..\\SlowedTTS.xml"));

    var reader = new XmlSerializer(typeof(Speak));
    var file = new StreamReader(filePath);
    var speakRequst = (Speak)reader.Deserialize(file);
    file.Close();

    speakRequst.Voice.Prosody.Text = text;
    speakRequst.Voice.Prosody.Rate = invertedRate.ToString("P");

    using var stringwriter = new StringWriter();
    var serializer = new XmlSerializer(typeof(Speak));
    serializer.Serialize(stringwriter, speakRequst);
    return stringwriter.ToString();
}
```

Finally, we can update the ```ConvertTextToSpeechAndSave``` implementation to load the XML and then call the ```SpeakSsmlAsync``` method. 

```csharp
private static async Task ConvertTextToSpeechAndSave(string text, float rate, BlockBlobClient blob, ExecutionContext context, ILogger log)
{
    log.LogInformation("Starting ConvertTextToSpeechAndSave");
    var xml = TextToSpeechSettingsXml(context, text, rate);

    //... more of the original implementation

    using(var speechSynthesizer = new SpeechSynthesizer(speechConfig, fileOutput))
    {
      // Change this line from speechSynthesizer.SpeakTextAsync(text);
      var speechSynthesisResult = await speechSynthesizer.SpeakSsmlAsync(xml);

      //... more of the original implementation

```

### Wrapping up 

The last piece of the puzzle is to invoke the ```ConvertTextToSpeechAndSave``` from the ```Run``` method and generate a response. The response will be some JSON containing the original input, the translation and the audio file URLS. 

```csharp
public class Response
{
    public string English { get; set; }

    public string Italian { get; set; }

    public string NormalSpeedUrl { get; set; }

    public string HalfSpeedUrl { get; set; }
}
```

Finally, the last half of the ```Run``` method should look like the following: 

```csharp
// Convert the translated text to slow speed audio.
await ConvertTextToSpeechAndSave(translatedText, 0.5f, halfSpeedAudioBlob, context, log);         
// Convert the translated text to normal speed audio.
await ConvertTextToSpeechAndSave(translatedText, 1.0f, normalSpeedAudioBlob, context, log);

// Build response.
var response = new Response
{
    English = englishInput,
    Italian = translatedText,
    NormalSpeedUrl = normalSpeedAudioBlob.Uri.ToString(),
    HalfSpeedUrl = halfSpeedAudioBlob.Uri.ToString()
};

return new HttpResponseMessage(HttpStatusCode.OK)
```


## Resources

- [Create a C# Function in Azure using Visual Studio Code](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-csharp?tabs=in-process)

- [Monitoring Azure Functions with Application Insights](https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring)
