# Hugo - the fast way to create static website

1. Create new site
  `hugo.exe new site site_name `

2. Create new page
  `hugo.exe new about.md`
  The new created page will be in content/ folder. 

3. Run hugo server to check contents locally. 
  `hugo.exe server -D`

4. Deploy

  `hugo.exe --theme=m10c --baseUrl="https://willisyi.github.io/website/"`
