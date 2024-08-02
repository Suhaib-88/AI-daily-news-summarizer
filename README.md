# AI news summarizer 
Finding relevant news articles and getting concise summaries can be time-consuming. This project aims to develop a tool that scrapes news articles from Google News, summarizes them using Groq, and emails the summaries to users.


# Code Overview (From the Jupyter Notebook):

## Import Libraries:
```
from GoogleNews import GoogleNews
import openai
import requests
import pandas as pd
from mailtrap import MailtrapClient, Mail, Address
```


## Perform news scraping from Google and extract the result into Pandas dataframe.
```
googlenews = GoogleNews(lang='en', region='US', period='1d', encode='utf-8')
googlenews.clear()
googlenews.search(keyword="<USER_SELECTED_TOPIC>")
news_result = googlenews.result(sort=True)
news_data_df = pd.DataFrame.from_dict(news_result)
```


## Gathering News Content text from links
```
ua = UserAgent()
news_data_df_with_text = []
for index, headers in news_data_df.iterrows():
    news_title = str(headers['title'])
    news_media = str(headers['media'])
    news_update = str(headers['date'])
    news_timestamp = str(headers['datetime'])
    news_description = str(headers['desc'])
    news_link = str(headers['link'])
    print(news_link)
    news_img = str(headers['img'])
    try:
        # html = requests.get(news_link).text
        html = requests.get(news_link, headers={'User-Agent':ua.chrome}, timeout=5).text
        text = fulltext(html)
        news_data_df_with_text.append([news_title, news_media, news_update, news_timestamp,
                                         news_description, news_link, news_img, text])
        print('Text Content Scraped')
    except:
        print('Text Content Scraped Error, Skipped')
        pass



news_data_with_text_df = pd.DataFrame(news_data_df_with_text, columns=['Title', 'Media', 'Update', 'Timestamp',
                                                                    'Description', 'Link', 'Image', 'Text'])
```

## Chunking news content into smaller and processable chunks

```
def break_up_file(tokens, chunk_size, overlap_size):
    if len(tokens) <= chunk_size:
        yield tokens
    else:
        chunk = tokens[:chunk_size]
        yield chunk
        yield from break_up_file(tokens[chunk_size-overlap_size:], chunk_size, overlap_size)

def break_up_file_to_chunks(stringname, chunk_size=2000, overlap_size=100):
    tokens = word_tokenize(stringname)
    return list(break_up_file(tokens, chunk_size, overlap_size))

```
## Generating per Articles summary:
```
for i, chunk in enumerate(chunks):
    print("Processing chunk " + str(i))
    prompt_request = "Summarize this news content: " + convert_to_detokenized_text(chunks[i])
    response = client.chat.completions.create(
        messages=[
        # Set an optional system message. This sets the behavior of the
        # assistant and can be used to provide specific instructions for
        # how it should behave throughout the conversation.
        {"role": "system", "content": "you are an Expert news summarizer AI assistant."},
        # Set a user message for the assistant to respond to.
        {
            "role": "user",
            "content": prompt_request,
        },
    ],

            model="llama-3.1-70b-versatile",
            temperature=.5, # Default is 1.
            max_tokens=500,
            top_p=1 # Default is 0.5.
    )
    print(response)
    prompt_response.append(response.choices[0].message.content.strip())


## Generating an overall news summary for the day:

response = client.chat.completions.create(
        messages=[
        # Set an optional system message. This sets the behavior of the
        # assistant and can be used to provide specific instructions for
        # how it should behave throughout the conversation.
        {"role": "system", "content": "you are an Expert news summarizer AI assistant."},
        # Set a user message for the assistant to respond to.
        {
            "role": "user",
            "content": prompt_request,
        },
    ],

            model="mixtral-8x7b-32768",
            temperature=.5, # Default is 1.
            max_tokens=500,
            top_p=1 # Default is 0.5.
    )
```
## Send Email via Mailtrap:
```
mail = Mail(
    sender=Address(email="your_email@example.com", name="Your Name"),
    to=[Address(email="recipient@example.com")],
    subject="GPT News Summary of Today!",
    html=f"<h3>Here is the news summary of GPT for today.</h3>{news_html}<br><br><h3>News Sources</h3>{news_html}"
)

client = MailtrapClient(token="your_mailtrap_api_token")
client.send(mail)
```