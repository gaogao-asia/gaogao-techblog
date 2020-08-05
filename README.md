# Gatsby HeroBlog
This blog is based on [GatsbyJS](https://www.gatsbyjs.org/) with a blog theme [HeroBlog](https://github.com/greglobinski/gatsby-starter-hero-blog)

## Remarks
- A dependency module `gatsby-plugin-styled-jsx-postcss` is not maintained anymore. So please do not update to latest. The latest modules causes a hanging issues on Netlify. Please see the detail.
https://github.com/gatsbyjs/gatsby/issues/21885

- Build time is quite long around serveral minutes after a post is published

## How to Post with Netlify CMS

1. Access to Netlify CMS
https://tech.gaogao.asia/admin/

2. Authenticate with your github account
You need to have an access previlege to this repository

3. Click a button 'New Post'

4. Click a button 'Publish now'

## How to Post with local gatsby

1. Install gatsby cli by `npm install -g gatsby-cli`

2. git clone this repository

3. Add a post under a dir `content/posts`
The folder name must be `{yyyy}-{MM}-{dd}--{path}` e.g. `content/posts/2020-08-03--test1`

4. Create an `index.md` under the dir
If you need a photos, please locate then under the post dir.

5. Check on local website by `gatsby develop` with `http://localhost:8000/`

6. `git commit` and `git push` to master branch

## Performance Tips
- https://blog.ojisan.io/gatsby-meet-netlify

