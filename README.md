# chatbot_csharp

## Slide 21
```
private async Task<string> GetQnAResponse(string question)
{
	using (var client = new HttpClient())
	using (var request = new HttpRequestMessage())
	{
		request.Method = HttpMethod.Post;
		request.RequestUri = new Uri(_config.GetValue<string>("QnAUri"));
		request.Content = new StringContent("{question:'" + question + "'}", Encoding.UTF8, "application/json");

		// The value of the header contains the string/text 'EndpointKey ' with the trailing space
		request.Headers.Add("Authorization", "EndpointKey " + _config.GetValue<string>("QnAEndpointKey"));

		var response = await client.SendAsync(request);
		var responseBody = await response.Content.ReadAsStringAsync();
		return JObject.Parse(responseBody)["answers"][0]["answer"].ToString();
	}
}
```

## Slide 22
```
var connector = new ConnectorClient(new Uri(turnContext.Activity.ServiceUrl), _config.GetValue<string>("MicrosoftAppId"), _config.GetValue<string>("MicrosoftAppPassword"));
var reply = (turnContext.Activity as Activity).CreateReply();
string userWords = turnContext.Activity.Text;
string predictionResult;

if (!string.IsNullOrWhiteSpace(userWords))
{
	// Get the answer from the QnA maker
	predictionResult = await GetQnAResponse(userWords);
	if (predictionResult != "No good match found in KB.")
	{
		reply.Text = (turnContext.Activity as Activity).CreateReply(predictionResult);
	}

	if (reply.Text.Length == 0)
	{
		reply.Text = "不好意思，機器人客服無法判斷您的意思，請重新說明您的問題";
	}
	
	await connector.Conversations.ReplyToActivityAsync(reply);
}			
```

## Slide 23
```
if (turnContext.Activity.ChannelId.ToLower() == "line")
{
	isRock.LineBot.Utility.ReplyMessage(reply.ReplyToId, reply.Text, _config.GetValue<string>("LineAccessToken"));
}
else
{
	await connector.Conversations.ReplyToActivityAsync(reply);
}
```

## Slide 32
```
requests
| where url endswith "generateAnswer"
| project timestamp, id, name, resultCode, duration
| parse name with *"/knowledgebases/"KbId"/generateAnswer"
| join kind= inner (
traces | extend id = operation_ParentId
) on id
| extend question = tostring(customDimensions['Question'])
| extend answer = tostring(customDimensions['Answer'])
| project KbId, timestamp, resultCode, duration, question, answer
```

## Slide 34
```
private void SaveData(string address)
{
	// Create WebRequest and set request uri string
	HttpWebRequest request = (HttpWebRequest)WebRequest.Create(address);
	// Http verb for the request
	request.Method = "POST";
	// Send request and get response
	HttpWebResponse response = (HttpWebResponse)request.GetResponse();
	// Close the response for new connections
	response.Close();
}
```

## Slide 35
```
private void CollectRequestData(ITurnContext<IMessageActivity> turnContext, string answer)
{
	// Convert UTC Time to Taipei Time
	DateTime timeUtc = DateTime.UtcNow;
	TimeZoneInfo taipeiZone = TimeZoneInfo.FindSystemTimeZoneById("Taipei Standard Time");
	DateTime taipeiTime = TimeZoneInfo.ConvertTimeFromUtc(timeUtc, taipeiZone);

	string scriptId = "GoogleScriptId";
	string address = $"https://docs.google.com/forms/d/e/{scriptId}/formResponse?";
	// DateTime
	address += "entry.1=" + HttpUtility.UrlEncode(taipeiTime.ToString("yyyyMMddHHmmss"), myEncoding);
	// Source
	address += "&entry.2=" + HttpUtility.UrlEncode(turnContext.Activity.ChannelId, myEncoding);
	// UserRequest
	address += "&entry.3=" + HttpUtility.UrlEncode(turnContext.Activity.Text, myEncoding);
	// BotResponse
	address += "&entry.4=" + HttpUtility.UrlEncode(answer, myEncoding);
	// UserId
	address += "&entry.5=" + HttpUtility.UrlEncode(turnContext.Activity.From.Id, myEncoding);
	// UserName
	address += "&entry.6=" + HttpUtility.UrlEncode(turnContext.Activity.From.Name, myEncoding);
	// Json
	var json = JsonConvert.SerializeObject(turnContext.Activity);
	address += "&entry.7=" + HttpUtility.UrlEncode(json, myEncoding);
	address += "&submit=Submit";
	SaveData(address);
}
```

## Slide 41
```
private async Task<Dictionary<string, string>> GetLUISPrediction(string text)
{
	using (var client = new HttpClient())
	using (var request = new HttpRequestMessage())
	{
		request.Method = HttpMethod.Get;
		string uri = _config.GetValue<string>("LuisEndpoint") + HttpUtility.UrlEncode(text, myEncoding);
		request.RequestUri = new Uri(uri);

		var response = await client.SendAsync(request);
		var responseBody = await response.Content.ReadAsStringAsync();
		var jsonObject = JObject.Parse(responseBody);
		string intent = jsonObject["intents"][0]["intent"].ToString();
		string entity = "但是我無法進一步分析" + intent;
		if (jsonObject.ContainsKey("compositeEntities"))
		{
			entity = jsonObject["compositeEntities"][0]["value"].ToString();
		}
		else
		{
			if (jsonObject["entities"].Count() > 0)
			{
				entity = jsonObject["entities"][0]["entity"].ToString();
			}
		}

		return new Dictionary<string, string>()
		{ {"Intent", intent},
		  {"Entity", entity}};
	}
}
```
