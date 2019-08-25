## Slide 26
```
private readonly IConfiguration _config;
public EchoBot(IConfiguration config)
{
	_config = config;
}

private async Task<string> GetQnAResponse(string question)
{
	using (var client = new HttpClient())
	using (var request = new HttpRequestMessage())
	{
		request.Method = HttpMethod.Post;
		request.RequestUri = new Uri(_config["QnAUri"]);
		request.Content = new StringContent("{question:'" + question + "'}", Encoding.UTF8, "application/json");

		// The value of the header contains the string/text 'EndpointKey ' with the trailing space
		request.Headers.Add("Authorization", "EndpointKey " + _config["QnAEndpointKey"]);

		var response = await client.SendAsync(request);
		var responseBody = await response.Content.ReadAsStringAsync();
		return JObject.Parse(responseBody)["answers"][0]["answer"].ToString();
	}
}
```

## Slide 27
```
var connector = new ConnectorClient(new Uri(turnContext.Activity.ServiceUrl), _config["MicrosoftAppId"], _config["MicrosoftAppPassword"]);
var reply = (turnContext.Activity as Activity).CreateReply();
string userWords = turnContext.Activity.Text;
string predictionResult;

if (!string.IsNullOrWhiteSpace(userWords))
{
	// Get the answer from the QnA maker
	predictionResult = await GetQnAResponse(userWords);
	if (predictionResult != "No good match found in KB.")
	{
		reply = (turnContext.Activity as Activity).CreateReply(predictionResult);
	}

	if (reply.Text.Length == 0)
	{
		reply.Text = "不好意思，機器人客服無法判斷您的意思，請重新說明您的問題";
	}
	
	await connector.Conversations.ReplyToActivityAsync(reply);
}			
```

## Slide 37
```
if (turnContext.Activity.ChannelId.ToLower() == "line")
{
	isRock.LineBot.Utility.ReplyMessage(reply.ReplyToId, reply.Text, _config["LineAccessToken"]);
}
else
{
	await connector.Conversations.ReplyToActivityAsync(reply);
}
```

## Slide 43
```
private async Task<Dictionary<string, string>> GetLUISPrediction(string text)
{
	using (var client = new HttpClient())
	using (var request = new HttpRequestMessage())
	{
		var queryString = HttpUtility.ParseQueryString(string.Empty);
		// Request headers
		client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", _config["LuisKey"]);
		// Request parameters
		queryString["verbose"] = "true";
		//queryString["staging"] = "{boolean}";
		var uri = _config["LuisUri"] + queryString;

		HttpResponseMessage response;
		byte[] byteData = Encoding.UTF8.GetBytes("\"" + text + "\"");
		using (var content = new ByteArrayContent(byteData))
		{
			response = await client.PostAsync(uri, content);
			var result = await response.Content.ReadAsStringAsync();
			var jsonObject = JObject.Parse(result);
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
}
```

```
var luisPredction = await GetLUISPrediction(userWords);
if (luisPredction["Intent"] != "None")
{
	predictionResult = "OK，你想要" + luisPredction["Intent"] + "，" + luisPredction["Entity"];
	reply.Text = predictionResult;
}
else
{
	// Get the answer from the QnA maker
	// Todo
}
```

## Slide 47
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

## Slide 49
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

## Slide 50
```
Encoding myEncoding = Encoding.GetEncoding("UTF-8");

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

## Slide 52
```
if (turnContext.Activity.ChannelId.ToLower() == "line")
{
	// LINE ButtonsTemplate 有字數限制
	// LINE Templates 手機上無法顯示
	string puretext = System.Text.RegularExpressions.Regex.Replace(reply.Text, "<.*?>", string.Empty);
	if (createButtons && puretext.Length <= 50)
	{
		var ButtonsTemplateMsg = new isRock.LineBot.ButtonsTemplate();
		ButtonsTemplateMsg.text = puretext + "請問有幫助到您嗎?";
		ButtonsTemplateMsg.title = "查詢回覆";
		var actions = new List<isRock.LineBot.TemplateActionBase>();
		actions.Add(new isRock.LineBot.MessageAction() { label = "<<很有用>>", text = "<<很有用>>" });
		actions.Add(new isRock.LineBot.MessageAction() { label = "<<普通>>", text = "<<普通>>" });
		actions.Add(new isRock.LineBot.MessageAction() { label = "<<再加強>>", text = "<<再加強>>" });
		ButtonsTemplateMsg.actions = actions;
		isRock.LineBot.Utility.ReplyTemplateMessage(reply.ReplyToId, ButtonsTemplateMsg, _config["LineAccessToken"]);
	}
	else
	{
		isRock.LineBot.Utility.ReplyMessage(reply.ReplyToId, reply.Text, _config["LineAccessToken"]);
	}
}
else
{
	if (createButtons)
	{
		reply = (turnContext.Activity as Activity).CreateReply(reply.Text + "\n\n請問有幫助到您嗎?");
		reply.SuggestedActions = new SuggestedActions()
		{
			Actions = new List<CardAction>()
			{
				new CardAction() { Title = "<<很有用>>", Type = ActionTypes.ImBack, Value = "<<很有用>>" },
				new CardAction() { Title = "<<普通>>", Type = ActionTypes.ImBack, Value = "<<普通>>" },
				new CardAction() { Title = "<<再加強>>", Type = ActionTypes.ImBack, Value = "<<再加強>>" }
			},
		};
	}
	await connector.Conversations.ReplyToActivityAsync(reply);
}

```

## Slide 54
```
private string ReadData(string source, string user)
{
	string sheetId = "GoogleSheetId";
	var address = $"https://spreadsheets.google.com/feeds/cells/{sheetId}/1/public/values?alt=json";

	//建立 WebRequest 並指定目標的 uri
	HttpWebRequest request = (HttpWebRequest)HttpWebRequest.Create(address);
	//指定 request 使用的 http verb
	request.Method = "GET";
	request.ContentType = "application/json; charset=utf-8";

	var timeString = "";
	//使用 GetResponse 方法將 request 送出
	using (HttpWebResponse response = (HttpWebResponse)request.GetResponse())
	//使用 GetResponseStream 方法從 server 回應中取得資料，stream 必須被關閉
	using (StreamReader streamreader = new StreamReader(response.GetResponseStream()))
	{
		timeString = streamreader.ReadToEnd();
	}
	var list = JObject.Parse(timeString)["feed"]["entry"];
	timeString = "";
	for (int i = list.Count() - 1; i >= 0; i--)
	{
		if (list[i]["content"]["$t"].ToString() == source && list[i + 3]["content"]["$t"].ToString() == user)
		{
			timeString = list[i - 1]["content"]["$t"].ToString();
			break;
		}
	}

	return timeString;
}
```

## Slide 55
```
private void CollectSurveyData(ITurnContext<IMessageActivity> turnContext, string result)
{
	string timeString = ReadData(turnContext.Activity.ChannelId, turnContext.Activity.From.Id);

	if (timeString.Length > 0)
	{
		string scriptId = "GoogleScriptId2";
		string address = $"https://docs.google.com/forms/d/e/{scriptId}/formResponse?";
		// DateTime
		address += "entry.1=" + HttpUtility.UrlEncode(timeString, myEncoding);
		// Source
		address += "&entry.2=" + HttpUtility.UrlEncode(turnContext.Activity.ChannelId, myEncoding);
		// UserId
		address += "&entry.3=" + HttpUtility.UrlEncode(turnContext.Activity.From.Id, myEncoding);
		// Remark
		address += "&entry.4=" + HttpUtility.UrlEncode(gradeDictionary.ContainsKey(result) ? gradeDictionary[result] : result, myEncoding);
		address += "&submit=Submit";
		SaveData(address);
	}
}
```

## Slide 56
```
static readonly Dictionary<string, string> gradeDictionary = new Dictionary<string, string>()
{{"<<很有用>>", "90" }, {"<<普通>>",  "50" }, {"<<再加強>>",  "10" }};
```

```
if (userWords.StartsWith("<<"))
{
	reply.Text = "謝謝您！歡迎繼續發問喔！";
	answerServey = true;
}
```

```
if (answerServey)
{
	CollectSurveyData(turnContext, userWords);
}
else
{
	CollectRequestData(turnContext, predictionResult);
}
```

```
=IFERROR(VLOOKUP(B2,'表單回應 2'!B:E,4,false),-99)
```












