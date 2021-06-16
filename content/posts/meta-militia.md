---
title: "Meta Militia, or Using a Custom Domain with GitHub Pages"
date: 2021-06-15T13:24:43+02:00
draft: false
---
How to: Using a custom domain (apex domain and _www_ subdomain) for a Hugo-generated page hosted on GitHub Pages.

# Introduction
What better first post than a meta post detailing the experience of getting my page running under my own domain in the first place. On the _true developer_-scale probably right below writing your own custom static site generator (in Rust or Kotlin, of course). If you're too cheap for the all-inclusive Wordpress experience, too vain to just use the _\<username\>.github.io_ domain, or too lazy to read the actual (very decent docs), this one's for you!

# Body
Hint: The instructions assume you already have your domain registered and a Hugo page lying around on your disk somewhere. In my examples, I will describe the setup with the domains _(www.) romanboehm.com_ and _rmnbhm.github.io_. If you want to follow along, exchange those with your own domain and your desired additional _.github.io_ domain.

Those are the steps needed to make it work:

1. Create a repository on GitHub called _rmnbhm.github.io_. You can actually name it however you want, I just follow GitHub Pages' convention here of using _\<username\>.github.io_. Added benefit: Your page will be available under that domain as well!
2. Navigate to your local git repository for the page's source code and add the remote repository as _origin_: 
    ```
    git remote add origin git@github.com:rmnbhm/rmnbhm.github.io.git
    ```
3. To allow for automatic builds and deployments create a folder in your repository's root called _.github_ which itself contains a folder called _workflows_. The latter then contains a file called _gh-pages.yml_ providing all the GitHub Actions steps to build and deploy the page:
    ```
    # .github/workflows/gh-pages.yml
    name: github pages

    on:
      push:
        branches:
          - main  # Set a branch to deploy
      pull_request:

    jobs:
      deploy:
        runs-on: ubuntu-20.04
        steps:
          - uses: actions/checkout@v2
            with:
              submodules: true  # Fetch Hugo themes (true OR recursive)
              fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

          - name: Setup Hugo
            uses: peaceiris/actions-hugo@v2
            with:
              hugo-version: 'latest'
              # extended: true

          - name: Build
            run: hugo --minify

          - name: Deploy
            uses: peaceiris/actions-gh-pages@v3
            if: github.ref == 'refs/heads/main'
            with:
              github_token: ${{ secrets.GITHUB_TOKEN }}
              publish_dir: ./public
    ``` 
4. Add a file called _CNAME_ to your _static_ folder. The file contains just the apex domain, like so:
    ```
    romanboehm.com
    ```
    This file will get included in your published files upon building the page, as described [here](https://gohugo.io/hosting-and-deployment/hosting-on-github/#use-a-custom-domain).
5. Push your page's source to GitHub:
    ```
    git push origin main
    ```
6. Navigate to your repository's _Pages_ settings on GitHub. If the initial actions run was succesful you should be able to pick _gh-pages_ as the _Source_ branch. Use _/_ (root) as the desired folder from which GitHub is supposed to source the page's content after a push.
7. Make sure the domain provided through the _CNAME_ file appears under the _Custom domain_ section. In my case that's _romanboehm.com_
8. In your DNS provider's (or registrar's) settings add the following *A* records, with the record's _name_ being the apex domain. The IP addresses belong to GitHub Pages (cf. the [GitHub Pages docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain)).
    ```
    romanboehm.com.	1	IN	A	185.199.111.153
    romanboehm.com.	1	IN	A	185.199.110.153
    romanboehm.com.	1	IN	A	185.199.109.153
    romanboehm.com.	1	IN	A	185.199.108.153
    ```
9. Then, add the following *CNAME* record.
    ```
    www.romanboehm.com.	1	IN	CNAME	rmnbhm.github.io.
    ```
    This will make your page available under the _www_ subdomain, too.

### Conclusion
That's the whole thing. You should verify your site is available under the desired apex, _www_, and _github.io_ (sub-) domains. Don't forget to update your config.toml to make use of the new base URL if you have any settings referencing the URL there.

