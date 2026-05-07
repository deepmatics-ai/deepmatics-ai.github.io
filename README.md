# Deepmatics Blog

This repository contains the source code for the Deepmatics blog, a Jekyll-powered site. The project is structured to separate content creation from theme management, making it easy to add new posts without modifying the site's underlying structure.

## Adding a New Post

To add a new post to the blog, follow these steps:

### 1. Create a New Post File

Navigate to the `_articles` directory and create a new Markdown file. The filename should be descriptive and use hyphens to separate words.

For example: `your-post-title.md`

### 2. Add Content and Front Matter

Open your new file and add the required YAML front matter at the top. The front matter defines the post's metadata, such as its title, date, and author. Below the front matter, you can write your post content using Markdown.

Use the following template for your post:

```markdown
---
layout: post
title: "Your Post Title Here"
date: YYYY-MM-DD
image: "/assets/images/your-image.jpg"  # Optional: Add an image for the post
excerpt: "A short, one-sentence summary of your post."
author: "Your Name"
tags: [Some, Relevant, Tags]
---

# Your Post Title

Start writing your blog post content here in Markdown.

## A Subheading

You can add more content, images, and code blocks as needed.
```

**Note:**

*   If you include an image, place the image file in the appropriate subfolder within the `assets` directory (e.g., `assets/images/`).

### 3. Rebuild the Site

After saving your new post file, you need to rebuild the Jekyll site for the changes to take effect. Run the following command in your terminal:

```bash
bundle exec jekyll build
```

Your new post will now appear on the main page.

## Project Structure

The repository is organized into two main areas:

*   **User Files (Content Creation):**
    *   `_articles/`: Contains all blog posts.
    *   `assets/`: Stores all images, CSS, and other static assets.
    *   `pages/`: Contains the site's static pages, such as "About Me."

*   **System Files (Theme and Layout):**
    *   `_app/`: Contains all theme-related files, including layouts, includes, and Sass files. These files are not typically edited when creating new content.
