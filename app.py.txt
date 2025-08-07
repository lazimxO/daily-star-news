from flask import Flask, render_template_string
import requests
from bs4 import BeautifulSoup
from nltk.tokenize import sent_tokenize
import nltk

app = Flask(__name__)

# Download NLTK punkt tokenizer on startup
nltk.download('punkt')

def summarize(text, max_sentences=2):
    sentences = sent_tokenize(text)
    return ' '.join(sentences[:max_sentences]) if sentences else text

def get_top_news():
    base_url = "https://www.thedailystar.net"
    response = requests.get(base_url)
    response.raise_for_status()
    soup = BeautifulSoup(response.text, 'html.parser')

    articles = []
    news_links = soup.select('div.view-content h3 a')[:10]

    for a in news_links:
        title = a.get_text(strip=True)
        link = a.get('href')
        if not link.startswith('http'):
            link = base_url + link

        try:
            article_resp = requests.get(link)
            article_resp.raise_for_status()
            article_soup = BeautifulSoup(article_resp.text, 'html.parser')
            paragraphs = article_soup.select('div.field-body p')
            full_text = ' '.join(p.get_text() for p in paragraphs)
            summary = summarize(full_text)
        except Exception:
            summary = "Summary not available."

        articles.append({'title': title, 'link': link, 'summary': summary})

    return articles

@app.route('/')
def home():
    try:
        news = get_top_news()
    except Exception:
        news = []

    html_template = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Top 10 Latest News - The Daily Star Bangladesh</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; background-color: #f9f9f9; }
            .news-item { margin-bottom: 30px; padding: 15px; border-bottom: 1px solid #ccc; background: white; border-radius: 5px; }
            h2 { font-size: 1.2em; margin-bottom: 8px; }
            p.summary { font-size: 1em; color: #333; }
            a { text-decoration: none; color: #007acc; }
            a:hover { text-decoration: underline; }
            h1 { color: #333; }
        </style>
    </head>
    <body>
        <h1>Top 10 Latest News - The Daily Star Bangladesh</h1>
        {% if news %}
            {% for article in news %}
                <div class="news-item">
                    <h2><a href="{{ article.link }}" target="_blank" rel="noopener noreferrer">{{ article.title }}</a></h2>
                    <p class="summary">{{ article.summary }}</p>
                </div>
            {% endfor %}
        {% else %}
            <p>No news to display at the moment.</p>
        {% endif %}
    </body>
    </html>
    """
    return render_template_string(html_template, news=news)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
