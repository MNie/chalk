---
layout: post
title: Bing Api Spell Check - Are you sure that you wrote it correctly?
description: "Thanks to BingApi Spell Check we can verify the spelling of a individual words as well as grammatical correctness of whole sentences. According to official documentation, we have the ability to even recognize the slang frequent mistakes in the text (eg. indeed, not inded etc). In this article we gonna give it a try."
thumb_image: "documentation/sample-image.jpg"
tags: [azure, fsharp, cognitiveservices]
---

Hi,

My next post I want to dedicate topics related to Azure. This time, I became interested in function, which by Azure'a is provided for us via Bing Api. This function is Spell Check.
What possibilities it gives us?
We can verify the spelling of a individual words as well as grammatical correctness of whole sentences. According to official documentation, we have the ability to recognize the slang frequent mistakes in the text (eg. indeed, not inded etc).
Unfortunately, the text sent/checked must be in English, specifically in the American variety. Soon there should be a option for other varieties of English (British, Canadian, Australian, the New Zealand, Indian) and also Chinese and Spanish.

We know what the api does, so let's see if it actually works!

Firstly, what we will need and in what format?
We need to send a POST request. The request should include the query type. There are two types. The first is called 'proof' explores the whole sentences and has the ability to improve their studying also based on the context surrounding sentences, checking grammar, any typos.
The second type is a 'spell' which checks for a single words or short sentences. This mode you can visualize as a search text correction provided by google/bing search engines. Text in this mode should consists maximum 9 words.

Apart from this we also need the application key, which we can generate on a Cognitive Services Microsoft subscription site: https://www.microsoft.com/cognitive-services/en-US/subscriptions
Another thing we must remember is to set ContentType message as: application/x-www-form-urlencoded.
The last point, the simplest, but most important is the information about how we pass the text?
We should pass it as a string assigned to the value of the Text in the body of the message we send. So, for example "Text = This is it"

Okay, we know from what we need to start, so it's time to go to the code. Because of all of the above steps I wrote following piece of code:

<script src="https://gist.github.com/MNie/318a3f0911ea6e6f0eee925906fac4ea.js"></script>

As You can see I wrote 2 methods. First loadFile responsible for loading the file from the specified location, I will need it to load the content of a text file.
The getSpellCheck function is responsible for forwarding the request to the Bing API with the appropriate parameters (type, api_key and text to check).
It is worth noting that text is passed to the function and then inside body of the message it is 'converted' to the format adopted by the function.
In the message header as mentioned earlier ContentType is set to correct value and in the OCP-APIM-Subscription-Key is our api_key.
At the end as a result of a function I return a deserialized json.

..but wait a moment, deserialized json? Result from the Spell Check function is in JSON form? How does it look like?
Api result is in the form of a serialized JSON, which looks as follows:

<script src="https://gist.github.com/MNie/ffe28b412c494ea8d3456525e5137041.js"></script>

Okay, we know how result looks like, but what the specific field / arrays means?
The '_type' means a function which we call. In our case it will have a value: "spellcheck", list 'flaggedTokens' is a collection of all the words that have been marked as invalid.
For each word you get the following information:
1. field 'offset' means the distance of a invalid word from a text beginning
2. 'token' this is our word
3. 'type' it's an error type
4. 'suggestions' which interested us the most, this is a list containing a list of proposals on which ones we can change our incorrect word.
Each suggestion contains a text that is proposed and the field 'score' which means its assessment, it is worth noting that the smaller the value in the 'score' is the suggestion is more accurate.

If you are interested in what algorithm is used to search for such suggestions and why proposals with lower scores are better, I strongly urge you to read following article about the BK-Tree algorithm: https://nullwords.wordpress.com/2013/03/13/the -BK-tree-and-data-structure-for-spell-checking /

Okay, so we know how the result from the api looks. Time to show the code corresponding to the structure of the result(JSON)!

<script src="https://gist.github.com/MNie/60aabde4006972f68ee84bbcd087b212.js"></script>

Well, at this moment we have a function that will return a proposal to amend our text, but we would also like to correct any invalid word/phrase in our text.
And here we move on to the next piece of code:

<script src="https://gist.github.com/MNie/dcce97d80d45266f238984809c434485.js"></script>

The getCorrectedText is responsible for converting the input text so that all the proposals generated by the api would been applied to the text.
To do this, I create a mutable variable, it will be overwritten at any proposal (yes I know this is not the most efficient solution, but to check if the function works fine it's the most adequate :))
Then I iterate on all of the words that were considered as incorrect, choose from them the best proposal and replace incorrect word, at the end I return the improved text and we're done!

That would be all when it comes to code and for what we need.
Regarding the same experience of using this function, it is certainly fun to check with which words "machine" works fine and how long he can decipher the word and suggest its improvement.
Eg. For word 'should' function fared so long until I finished on the 'sld' and at this point it has no idea what to offer.
I would recommend anyone to try this feature, in particular in 'proof' mode where we can check a larger piece of text, an interesting attempt may be sent to api lyrics/email/conversation where it could be a lot of informal phrases which will allow us to fully check api power and try it on our own.
Here we can see an example of the result of passing the incorrect text by api together with proposals for improvement:
<script src="https://gist.github.com/MNie/78ff84eaae6c97bc8461af61c52c2e5e.js"></script>
![Image of bad text](https://mnie.github.com/img/\AzureBingSpellCheck/result.png)
As we can see api has detected 6 from 14 incorrect words/phrases it's not a outstanding result, but it also has spotted a couple of errors with spaces and small/big letters of words. So in my opinion it works rather good.

All code could be found [here](https://github.com/MNie/AzureBingSpellCheck).

Thank you for reading :)