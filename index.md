---
layout: default
title: Deepmatics
---

{% assign sorted_articles = site.articles | sort: 'date' | reverse %}

{% assign all_tags = "" %}
{% for post in sorted_articles %}
    {% for tag in post.tags %}
        {% unless all_tags contains tag %}
            {% if all_tags == "" %}
                {% assign all_tags = tag %}
            {% else %}
                {% assign all_tags = all_tags | append: "," | append: tag %}
            {% endif %}
        {% endunless %}
    {% endfor %}
{% endfor %}
{% assign tag_list = all_tags | split: "," | sort %}

<div class="blog-toolbar">
    <div class="toolbar-top">
        <input type="text" class="search-input" id="search-input" placeholder="Search posts..." />
        <button class="toolbar-btn" id="view-toggle" title="Toggle view">Grid</button>
        <button class="toolbar-btn" id="sort-toggle" title="Toggle sort order">Newest</button>
    </div>
    <div class="toolbar-tags">
        <span class="tag-pill active" data-tag="all">All</span>
        {% for tag in tag_list %}
        <span class="tag-pill" data-tag="{{ tag }}">{{ tag }}</span>
        {% endfor %}
    </div>
    <div class="toolbar-bottom">
        <span class="post-count" id="post-count">Showing {{ sorted_articles.size }} of {{ sorted_articles.size }} posts</span>
    </div>
</div>

<div class="blog-posts-container grid-view" id="posts-container">
    {% for post in sorted_articles %}
    <div class="blog-card" data-tags="{{ post.tags | join: ',' }}" data-date="{{ post.date | date: '%Y-%m-%d' }}">
        <img src="{{ post.image | relative_url }}" alt="{{ post.title }}" class="card-image" />
        <div class="card-content">
            <div class="card-meta">
                <span class="card-date">{{ post.date | date: "%b %d, %Y" }}</span>
                <div class="card-tags">
                    {% for tag in post.tags %}
                    <span class="card-tag">{{ tag }}</span>
                    {% endfor %}
                </div>
            </div>
            <h3 class="card-title">
                <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
            </h3>
            <p class="card-description">
                {% if post.excerpt %}
                    {{ post.excerpt | strip_html | truncatewords: 25 }}
                {% else %}
                    {{ post.content | strip_html | truncatewords: 25 }}
                {% endif %}
            </p>
            <a href="{{ post.url | relative_url }}" class="card-link">Read More</a>
        </div>
    </div>
    {% endfor %}
</div>

<div class="no-results" id="no-results">
    No posts match your filters. Try a different search or tag.
</div>

<script>
(function() {
    var container = document.getElementById('posts-container');
    var cards = Array.from(container.querySelectorAll('.blog-card'));
    var searchInput = document.getElementById('search-input');
    var viewToggle = document.getElementById('view-toggle');
    var sortToggle = document.getElementById('sort-toggle');
    var tagPills = document.querySelectorAll('.tag-pill');
    var postCount = document.getElementById('post-count');
    var noResults = document.getElementById('no-results');
    var total = cards.length;

    var activeTag = 'all';
    var sortAsc = false;
    var isListView = false;

    if (localStorage.getItem('blog-view') === 'list') {
        isListView = true;
        container.classList.remove('grid-view');
        container.classList.add('list-view');
        viewToggle.textContent = 'List';
    }

    function filterAndRender() {
        var query = searchInput.value.toLowerCase().trim();
        var visible = 0;

        cards.forEach(function(card) {
            var tags = card.getAttribute('data-tags').split(',');
            var title = card.querySelector('.card-title').textContent.toLowerCase();
            var desc = card.querySelector('.card-description').textContent.toLowerCase();

            var tagMatch = activeTag === 'all' || tags.indexOf(activeTag) !== -1;
            var searchMatch = !query || title.indexOf(query) !== -1 || desc.indexOf(query) !== -1;

            if (tagMatch && searchMatch) {
                card.classList.remove('hidden');
                visible++;
            } else {
                card.classList.add('hidden');
            }
        });

        postCount.textContent = 'Showing ' + visible + ' of ' + total + ' posts';
        noResults.classList.toggle('visible', visible === 0);
    }

    function sortCards() {
        var sorted = cards.slice().sort(function(a, b) {
            var da = a.getAttribute('data-date');
            var db = b.getAttribute('data-date');
            return sortAsc ? da.localeCompare(db) : db.localeCompare(da);
        });
        sorted.forEach(function(card) {
            container.appendChild(card);
        });
    }

    tagPills.forEach(function(pill) {
        pill.addEventListener('click', function() {
            tagPills.forEach(function(p) { p.classList.remove('active'); });
            pill.classList.add('active');
            activeTag = pill.getAttribute('data-tag');
            filterAndRender();
        });
    });

    searchInput.addEventListener('input', filterAndRender);

    viewToggle.addEventListener('click', function() {
        isListView = !isListView;
        container.classList.toggle('grid-view', !isListView);
        container.classList.toggle('list-view', isListView);
        viewToggle.textContent = isListView ? 'List' : 'Grid';
        localStorage.setItem('blog-view', isListView ? 'list' : 'grid');
    });

    sortToggle.addEventListener('click', function() {
        sortAsc = !sortAsc;
        sortToggle.textContent = sortAsc ? 'Oldest' : 'Newest';
        sortCards();
    });
})();
</script>
