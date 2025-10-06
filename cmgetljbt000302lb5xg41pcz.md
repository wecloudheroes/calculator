---
title: "From Clone to Your Own — How I Turned Someone’s GitHub Repo into Mine"
datePublished: Mon Oct 06 2025 07:39:27 GMT+0000 (Coordinated Universal Time)
cuid: cmgetljbt000302lb5xg41pcz
slug: from-clone-to-your-own-how-i-turned-someones-github-repo-into-mine
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1759736191463/6d12470e-730d-4c08-8a75-53fbe8c2c6ec.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1759736349270/4400d5c6-8f8c-4729-b5be-234130697cab.png
tags: github, git, clone, pr, branching-strategies

---

Have you ever found an awesome GitHub project that you wanted to explore, improve, or make your own version of?  
That happened to me recently. I cloned someone’s repository into my VS Code setup, made some cool changes…  
and then realized — “Wait! If I push this, it’ll affect *their* repo!” 😅

So I needed a clean way to keep all my changes in **my own GitHub repository** without messing up the original one.  
If you’ve ever been there — don’t worry. Follow these simple steps, and you’ll have your own version up and running in no time 🚀

---

## 🧭 Step 1: Clone the Repository

First, clone the original repository you liked:

```plaintext
git clone https://github.com/originaluser/their-repo.git
```

Open it in **VS Code**:

```plaintext
cd their-repo
code .
```

Now you can explore the project, modify it, and make it truly yours!

---

## 🧹 Step 2: Disconnect from the Original Repo

By default, your cloned folder is still linked to the original repo (called *origin*).  
Let’s break that link:

```plaintext
git remote remove origin
```

To confirm it’s gone:

```plaintext
git remote -v
```

It should show nothing — that means your local copy is now independent.

---

## 🧱 Step 3: Create Your Own Repo on GitHub

Go to 👉 [https://github.com/new](https://github.com/new)

* Give it a name (say, `myproject`)
    
* Keep it empty (no README or .gitignore)
    
* Click **Create repository**
    

You’ll now see instructions like this:

```plaintext
git remote add origin https://github.com/yourusername/myproject.git
git branch -M main
git push -u origin main
```

We’ll use those next.

---

## 🔗 Step 4: Connect to Your New Repo

Back in VS Code terminal:

```plaintext
git remote add origin https://github.com/yourusername/myproject.git
git branch -M main
```

Now your local folder is linked to *your own* GitHub repository.

---

## 🚀 Step 5: Push the Code to Your Repo

If you try to push and see this:

```plaintext
remote: Invalid username or token.
fatal: Authentication failed
```

Don’t worry — GitHub now uses **Personal Access Tokens (PATs)** instead of passwords.

Go to  
👉 [https://github.com/settings/tokens](https://github.com/settings/tokens)  
Create a new token with:

* `repo` permission
    
* Copy the token and use it as your password when Git asks.
    

Then push:

```plaintext
git push -u origin main
```

✅ Done! You now have your own independent copy of the project on GitHub.

---

## 🧭 Bonus Tip: Keep the Original Repo as “Upstream” (Optional)

If you want to occasionally pull updates from the original project:

```plaintext
git remote add upstream https://github.com/originaluser/their-repo.git
git fetch upstream
git merge upstream/main
```

That way, you stay up to date without losing your changes.

---

## 🎉 That’s It!

And that’s how I took someone’s repo, learned from it, customized it, and made it mine — safely and cleanly.

Whether you’re experimenting, learning, or building something new, this small workflow makes a big difference.  
So go ahead — explore, clone, create, and share your own version with the world 🌍💡

---

### 💬 What’s next?

You can now:

* Update your `README.md` with your own name and purpose
    
* Add a license if it’s your new project
    
* Continue committing and pushing your updates confidently