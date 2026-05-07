---
layout: default
title: Deepmatics
---

<div class="blog-cards-container">
    {% assign sorted_articles = site.articles | sort: 'date' | reverse %}
    {% for post in sorted_articles %}
    <div class="blog-card">
        <img src="{{ post.image | relative_url }}" alt="{{ post.title }}" class="card-image" />
        <div class="card-content">
            <h3 class="card-title">{{ post.title }}</h3>
            <p class="card-description">
                {% if post.excerpt %}
                    {{ post.excerpt | strip_html | truncatewords: 20 }}
                {% else %}
                    {{ post.content | strip_html | truncatewords: 20 }}
                {% endif %}
            </p>
            <a href="{{ post.url | relative_url }}" class="card-link">Read More</a>
        </div>
    </div>
    {% endfor %}
</div>