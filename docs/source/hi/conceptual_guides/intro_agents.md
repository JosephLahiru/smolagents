<!--Copyright 2024 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.

-->
# Agents का परिचय

## 🤔 Agents क्या हैं?

AI का उपयोग करने वाली किसी भी कुशल प्रणाली को LLM को वास्तविक दुनिया तक किसी प्रकार की पहुंच प्रदान करने की आवश्यकता होगी: उदाहरण के लिए बाहरी जानकारी प्राप्त करने के लिए एक खोज टूल को कॉल करने की संभावना, या किसी कार्य को हल करने के लिए कुछ प्रोग्राम पर कार्य करने की। दूसरे शब्दों में, LLM में ***agency*** होनी चाहिए। एजेंटिक प्रोग्राम LLM के लिए बाहरी दुनिया का प्रवेश द्वार हैं।

> [!TIP]
> AI Agents वे **प्रोग्राम हैं जहां LLM आउटपुट वर्कफ़्लो को नियंत्रित करते हैं**।

LLM का उपयोग करने वाली कोई भी प्रणाली LLM आउटपुट को कोड में एकीकृत करेगी। कोड वर्कफ़्लो पर LLM के इनपुट का प्रभाव सिस्टम में LLM की एजेंसी का स्तर है।

ध्यान दें कि इस परिभाषा के साथ, "agent" एक अलग, 0 या 1 परिभाषा नहीं है: इसके बजाय, "agency" एक निरंतर स्पेक्ट्रम पर विकसित होती है, जैसे-जैसे आप अपने वर्कफ़्लो पर LLM को अधिक या कम शक्ति देते हैं।

नीचे दी गई तालिका में देखें कि कैसे एजेंसी विभिन्न प्रणालियों में भिन्न हो सकती है:

| एजेंसी स्तर | विवरण | इसे क्या कहा जाता है | उदाहरण पैटर्न |
|------------|---------|-------------------|----------------|
| ☆☆☆ | LLM आउटपुट का प्रोग्राम प्रवाह पर कोई प्रभाव नहीं | सरल प्रोसेसर | `process_llm_output(llm_response)` |
| ★☆☆ | LLM आउटपुट if/else स्विच निर्धारित करता है | राउटर | `if llm_decision(): path_a() else: path_b()` |
| ★★☆ | LLM आउटपुट फंक्शन एक्जीक्यूशन निर्धारित करता है | टूल कॉलर | `run_function(llm_chosen_tool, llm_chosen_args)` |
| ★★★ | LLM आउटपुट पुनरावृत्ति और प्रोग्राम की निरंतरता को नियंत्रित करता है | मल्टी-स्टेप एजेंट | `while llm_should_continue(): execute_next_step()` |
| ★★★ | एक एजेंटिक वर्कफ़्लो दूसरे एजेंटिक वर्कफ़्लो को शुरू कर सकता है | मल्टी-एजेंट | `if llm_trigger(): execute_agent()` |

मल्टी-स्टेप agent की यह कोड संरचना है:

```python
memory = [user_defined_task]
while llm_should_continue(memory): # यह लूप मल्टी-स्टेप भाग है
    action = llm_get_next_action(memory) # यह टूल-कॉलिंग भाग है
    observations = execute_action(action)
    memory += [action, observations]
```

यह एजेंटिक सिस्टम एक लूप में चलता है, प्रत्येक चरण में एक नई क्रिया को शुरू करता है (क्रिया में कुछ पूर्व-निर्धारित *tools* को कॉल करना शामिल हो सकता है जो केवल फंक्शंस हैं), जब तक कि उसके अवलोकन से यह स्पष्ट न हो जाए कि दिए गए कार्य को हल करने के लिए एक संतोषजनक स्थिति प्राप्त कर ली गई है।

## ✅ Agents का उपयोग कब करें / ⛔ कब उनसे बचें

Agents तब उपयोगी होते हैं जब आपको किसी ऐप के वर्कफ़्लो को निर्धारित करने के लिए LLM की आवश्यकता होती है। लेकिन वे अक्सर जरूरत से ज्यादा होते हैं। सवाल यह है कि, क्या मुझे वास्तव में दिए गए कार्य को कुशलतापूर्वक हल करने के लिए वर्कफ़्लो में लचीलेपन की आवश्यकता है?
यदि पूर्व-निर्धारित वर्कफ़्लो बहुत बार विफल होता है, तो इसका मतलब है कि आपको अधिक लचीलेपन की आवश्यकता है।

आइए एक उदाहरण लेते हैं: मान लीजिए आप एक ऐप बना रहे हैं जो एक सर्फिंग ट्रिप वेबसाइट पर ग्राहक अनुरोधों को संभालता है।

आप पहले से जान सकते हैं कि अनुरोध 2 में से किसी एक श्रेणी में आएंगे (उपयोगकर्ता की पसंद के आधार पर), और आपके पास इन 2 मामलों में से प्रत्येक के लिए एक पूर्व-निर्धारित वर्कफ़्लो है।

1. ट्रिप के बारे में कुछ जानकारी चाहिए? ⇒ उन्हें अपने नॉलेज बेस में खोज करने के लिए एक सर्च बार तक पहुंच दें
2. सेल्स टीम से बात करना चाहते हैं? ⇒ उन्हें एक संपर्क फॉर्म में टाइप करने दें।

यदि वह निर्धारणात्मक वर्कफ़्लो सभी प्रश्नों के लिए फिट बैठता है, तो बेशक बस सब कुछ कोड करें! यह आपको एक 100% विश्वसनीय सिस्टम देगा और एलएलएम द्वारा अनपेक्षित कार्यप्रवाह में हस्तक्षेप करने से त्रुटियों का कोई जोखिम नहीं होगा। साधारणता और मजबूती के लिए, सलाह दी जाती है कि एजेंटिक व्यवहार का उपयोग न किया जाए।

लेकिन क्या होगा अगर वर्कफ़्लो को पहले से इतनी अच्छी तरह से निर्धारित नहीं किया जा सकता?

उदाहरण के लिए, एक उपयोगकर्ता पूछना चाहता है: `"मैं सोमवार को आ सकता हूं, लेकिन मैं अपना पासपोर्ट भूल गया जिससे मुझे बुधवार तक देर हो सकती है, क्या आप मुझे और मेरी चीजों को मंगलवार सुबह सर्फ करने ले जा सकते हैं, क्या मुझे कैंसलेशन इंश्योरेंस मिल सकता है?"` यह प्रश्न कई कारकों पर निर्भर करता है, और शायद ऊपर दिए गए पूर्व-निर्धारित मानदंडों में से कोई भी इस अनुरोध के लिए पर्याप्त नहीं होगा।

यदि पूर्व-निर्धारित वर्कफ़्लो बहुत बार विफल होता है, तो इसका मतलब है कि आपको अधिक लचीलेपन की आवश्यकता है।

यहीं पर एक एजेंटिक सेटअप मदद करता है।

ऊपर दिए गए उदाहरण में, आप बस एक मल्टी-स्टेप agent बना सकते हैं जिसके पास मौसम पूर्वानुमान के लिए एक मौसम API, यात्रा की दूरी जानने के लिए के लिए Google Maps API, एक कर्मचारी उपलब्धता डैशबोर्ड और आपके नॉलेज बेस पर एक RAG सिस्टम तक पहुंच है।

हाल ही तक, कंप्यूटर प्रोग्राम पूर्व-निर्धारित वर्कफ़्लो तक सीमित थे, if/else स्विच का
ढेर लगाकार जटिलता को संभालने का प्रयास कर रहे थे। वे बेहद संकीर्ण कार्यों पर केंद्रित थे, जैसे "इन संख्याओं का योग निकालें" या "इस ग्राफ़ में सबसे छोटा रास्ता खोजें"। लेकिन वास्तव में, अधिकांश वास्तविक जीवन के कार्य, जैसे ऊपर दिया गया हमारा यात्रा उदाहरण, पूर्व-निर्धारित वर्कफ़्लो में फिट नहीं होते हैं। एजेंटिक सिस्टम प्रोग्राम के लिए वास्तविक दुनिया के कार्यों की विशाल दुनिया खोलते हैं!

## क्यों `smolagents`?

कुछ लो-लेवल एजेंटिक उपयोग के मामलों के लिए, जैसे चेन या राउटर, आप सभी कोड खुद लिख सकते हैं। आप इस तरह से बहुत बेहतर होंगे, क्योंकि यह आपको अपने सिस्टम को बेहतर ढंग से नियंत्रित और समझने की अनुमति देगा।

लेकिन जैसे ही आप अधिक जटिल व्यवहारों की ओर बढ़ते हैं जैसे कि LLM को एक फ़ंक्शन कॉल करने देना (यह "tool calling" है) या LLM को एक while लूप चलाने देना ("multi-step agent"), कुछ एब्सट्रैक्शन्स की आवश्यकता होती है:
- टूल कॉलिंग के लिए, आपको एजेंट के आउटपुट को पार्स करने की आवश्यकता होती है, इसलिए इस आउटपुट को एक पूर्व-निर्धारित प्रारूप की आवश्यकता होती है जैसे "विचार: मुझे 'get_weather' टूल कॉल करना चाहिए। क्रिया: get_weather(Paris)।", जिसे आप एक पूर्व-निर्धारित फ़ंक्शन के साथ पार्स करते हैं, और LLM को दिए गए सिस्टम प्रॉम्प्ट को इस प्रारूप के बारे में सूचित करना चाहिए।
- एक मल्टी-स्टेप एजेंट के लिए जहां LLM आउटपुट लूप को निर्धारित करता है, आपको पिछले लूप इटरेशन में क्या हुआ इसके आधार पर LLM को एक अलग प्रॉम्प्ट देने की आवश्यकता होती है: इसलिए आपको किसी प्रकार की मेमोरी की आवश्यकता होती है।

इन दो उदाहरणों के साथ, हमने पहले ही कुछ चीजों की आवश्यकता का पता लगा लिया:

- बेशक, एक LLM जो सिस्टम को पावर देने वाले इंजन के रूप में कार्य करता है
- एजेंट द्वारा एक्सेस किए जा सकने वाले टूल्स की एक सूची
- एक पार्सर जो LLM आउटपुट से टूल कॉल को निकालता है
- एक सिस्टम प्रोम्प्ट जो पार्सर के साथ सिंक्रनाइज़ होता है
- एक मेमोरी

लेकिन रुकिए, चूंकि हम निर्णयों में LLM को जगह देते हैं, निश्चित रूप से वे गलतियां करेंगे: इसलिए हमें एरर लॉगिंग और पुनः प्रयास तंत्र की आवश्यकता है।

ये सभी तत्व एक अच्छे कामकाजी सिस्टम बनाने के लिए एक-दूसरे से घनिष्ठ रूप से जुड़े हुए हैं। यही कारण है कि हमने तय किया कि इन सभी चीजों को एक साथ काम करने के लिए बुनियादी निर्माण ब्लॉक्स की आवश्यकता है।

## कोड Agents

एक मल्टी-स्टेप एजेंट में, प्रत्येक चरण पर, LLM बाहरी टूल्स को कुछ कॉल के रूप में एक क्रिया लिख सकता है। इन क्रियाओं को लिखने के लिए एक सामान्य स्वरूप (Anthropic, OpenAI और कई अन्य द्वारा उपयोग किया जाता है) आमतौर पर "टूल्स के नाम और उपयोग करने के लिए तर्कों के JSON के रूप में क्रियाएं लिखने" के विभिन्न रूप होते हैं, जिन्हें आप फिर पार्स करते हैं यह जानने के लिए कि कौन सा टूल किन तर्कों के साथ निष्पादित करना है"।

[कई](https://huggingface.co/papers/2402.01030) [शोध](https://huggingface.co/papers/2411.01747) [पत्रों](https://huggingface.co/papers/2401.00812) ने दिखाया है कि कोड में टूल कॉलिंग LLM का होना बहुत बेहतर है।

इसका कारण बस यह है कि *हमने अपनी कोड भाषाओं को विशेष रूप से कंप्यूटर द्वारा किए गए कार्यों को व्यक्त करने का सर्वोत्तम संभव तरीका बनाने के लिए तैयार किया*। यदि JSON स्निपेट्स बेहतर अभिव्यक्ति होते, तो JSON शीर्ष प्रोग्रामिंग भाषा होती और प्रोग्रामिंग नरक में होती।

नीचे दी गई छवि, [Executable Code Actions Elicit Better LLM Agents](https://huggingface.co/papers/2402.01030) से ली गई है, जो कोड में क्रियाएं लिखने के कुछ फायदे दर्शाती है:

<img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/code_vs_json_actions.png">

JSON जैसे स्निपेट्स की बजाय कोड में क्रियाएं लिखने से बेहतर प्राप्त होता है:

- **कम्पोजेबिलिटी:** क्या आप JSON क्रियाओं को एक-दूसरे के भीतर नेस्ट कर सकते हैं, या बाद में पुन: उपयोग करने के लिए JSON क्रियाओं का एक सेट परिभाषित कर सकते हैं, उसी तरह जैसे आप बस एक पायथन फंक्शन परिभाषित कर सकते हैं?
- **ऑब्जेक्ट प्रबंधन:** आप `generate_image` जैसी क्रिया के आउटपुट को JSON में कैसे स्टोर करते हैं?
- **सामान्यता:** कोड को सरल रूप से कुछ भी व्यक्त करने के लिए बनाया गया है जो आप कंप्यूटर से करवा सकते हैं।
- **LLM प्रशिक्षण डेटा में प्रतिनिधित्व:** बहुत सारी गुणवत्तापूर्ण कोड क्रियाएं पहले से ही LLM के ट्रेनिंग डेटा में शामिल हैं जिसका मतलब है कि वे इसके लिए पहले से ही प्रशिक्षित हैं!