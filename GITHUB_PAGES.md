# GitHub Pages Static Hosting

To maintain instant access across both your workstation and mobile devices without manually copying files, you can host Splice securely for free using GitHub Pages.

1. Ensure your main application file is named `index.html` in your repository.
2. Navigate to your repository **Settings** -> **Pages**.
3. Under **Build and deployment**, set the source to **Deploy from a branch** and select `main` (or `master`).
4. Click **Save**. 

Your instance will be live at `https://<your-username>.github.io/<repo-name>/` over a secure HTTPS connection. Note: HTTPS is strictly required by modern mobile browsers to grant microphone access.
