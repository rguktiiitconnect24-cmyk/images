# 📸 Profile Photo Upload Integration Guide

This guide explains how to integrate the **Picser** image hosting solution into your own application to handle user profile photo uploads.

## How it works

Since you don't want to expose your GitHub Token in your frontend code (React, Vue, HTML), the most secure way to handle uploads is:
1. **Frontend**: The user selects a profile photo and your frontend sends it to your app's backend.
2. **Backend**: Your backend securely holds your GitHub Token and forwards the image to the Picser API.
3. **Save URL**: Picser returns a lightning-fast CDN URL, which you can then save to your database as the user's profile picture.

---

## 1. The Frontend (React/Next.js Example)

Here is a simple React component that allows a user to select an image, previews it, and uploads it to your backend.

```jsx
import { useState } from 'react';

export default function ProfilePhotoUploader() {
  const [photoPreview, setPhotoPreview] = useState(null);
  const [isUploading, setIsUploading] = useState(false);

  const handleFileChange = async (event) => {
    const file = event.target.files[0];
    if (!file) return;

    // Show a local preview to the user instantly
    setPhotoPreview(URL.createObjectURL(file));

    // Upload the file
    setIsUploading(true);
    const formData = new FormData();
    formData.append("profile_image", file);

    try {
      // Send the file to YOUR app's backend route
      const response = await fetch("/api/upload-profile-photo", {
        method: "POST",
        body: formData,
      });

      const data = await response.json();
      
      if (data.success) {
        alert("Profile photo updated successfully!");
        console.log("New Photo URL:", data.url);
        // You would typically update your user's state/context here with data.url
      } else {
        alert("Upload failed.");
      }
    } catch (error) {
      console.error("Error uploading:", error);
    } finally {
      setIsUploading(false);
    }
  };

  return (
    <div style={{ display: "flex", flexDirection: "column", alignItems: "center" }}>
      <h3>Update Profile Picture</h3>
      
      <div style={{ width: 100, height: 100, borderRadius: "50%", overflow: "hidden", border: "2px solid #ccc" }}>
        {photoPreview ? (
          <img src={photoPreview} alt="Preview" style={{ width: "100%", height: "100%", objectFit: "cover" }} />
        ) : (
          <div style={{ width: "100%", height: "100%", backgroundColor: "#eee" }} />
        )}
      </div>

      <input 
        type="file" 
        accept="image/png, image/jpeg, image/webp" 
        onChange={handleFileChange} 
        disabled={isUploading}
        style={{ marginTop: "1rem" }}
      />
      
      {isUploading && <p>Uploading to CDN...</p>}
    </div>
  );
}
```

---

## 2. The Backend API Route (Next.js Example)

This is the secure part of your application (e.g., `app/api/upload-profile-photo/route.js` in Next.js). It takes the image from your frontend and forwards it to the Picser API using your hidden GitHub credentials.

```javascript
// Next.js Route Handler Example (app/api/upload-profile-photo/route.js)
import { NextResponse } from "next/server";

export async function POST(request) {
  try {
    const incomingFormData = await request.formData();
    const file = incomingFormData.get("profile_image");

    if (!file) {
      return NextResponse.json({ error: "No file provided" }, { status: 400 });
    }

    // 1. Prepare the payload for Picser
    const picserFormData = new FormData();
    picserFormData.append("file", file);
    
    // We get these from our secure environment variables!
    picserFormData.append("github_token", process.env.GITHUB_TOKEN);
    picserFormData.append("github_owner", process.env.GITHUB_OWNER);
    picserFormData.append("github_repo", process.env.GITHUB_REPO);

    // 2. Send the image to the Picser Hosted API
    const response = await fetch("https://picser.pages.dev/api/public-upload", {
      method: "POST",
      body: picserFormData,
    });

    const data = await response.json();

    if (data.success) {
      // Use the raw commit URL for instant display (jsDelivr can take a minute to cache)
      const permanentCdnUrl = data.data.urls.raw_commit; 

      // 3. Save `permanentCdnUrl` to your Database (MongoDB, PostgreSQL, etc.)
      
      // 4. Return the new URL back to the frontend
      return NextResponse.json({ success: true, url: permanentCdnUrl });
    } else {
      return NextResponse.json({ error: "Picser upload failed" }, { status: 500 });
    }
  } catch (error) {
    return NextResponse.json({ error: "Internal Server Error" }, { status: 500 });
  }
}
```

## 3. Environment Variables

In your actual app's `.env.local` file, make sure you have the exact same variables we just set up:

```env
GITHUB_TOKEN=your_github_personal_access_token_here
GITHUB_OWNER=rguktiiitconnect24-cmyk
GITHUB_REPO=images
```
