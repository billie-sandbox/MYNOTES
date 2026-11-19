---
created: 2026-11-19
topics:
  - "[[YouTube]]"
  - "[[Livestreaming]]"
  - "[[OBS]]"
---

> [!summary]
> There are two layers. On YouTube itself, creators can't restyle chat; the only things they control are custom emoji, member badges and chat modes. On stream, the chat overlay can be styled any way you want: load the popout chat as an OBS browser source and add custom CSS, or use a chat-overlay tool (Social Stream Ninja, Streamlabs/StreamElements widgets) that reads the chat and redraws it.

## 1. What YouTube lets the creator change (on youtube.com)

The look of chat for viewers is fixed. Creator settings only touch these:

- **Custom emoji**: none exist by default; upload them under Memberships → *Your badges and emoji*. Each set has a family name members type to autocomplete. Every member level gets every emoji.
- **Member badges**: 8 badges by membership length. Use the defaults or upload your own (JPEG/PNG, under 1 MB, shown at 16×16 px in chat).
- **Built-in badges and colors**: owner name gets a yellow highlight, moderators a blue name and wrench icon, verified a check. Not configurable.
- **Super Chat / Super Stickers**: the amount sets the tier, and the tier sets the highlight color, max message length and how long it stays pinned in the ticker. Colors run roughly blue (low) → green → yellow → orange → magenta → red (highest). Top tier ($500) can stay pinned for up to 5 hours. Exact cutoffs differ by currency; check YouTube's Help Center.
- **Chat modes**: members-only chat, subscribers-only, slow mode (doesn't apply to members), live chat replay on/off. New members get a green "New member" message.

Viewers can only pick Top chat vs Live chat, and light/dark theme.

## 2. Styling chat on the stream (overlay)

### Option A: popout chat + OBS custom CSS (free, DIY)

1. In chat, ⋮ → **Popout chat**. Copy the URL (it must be `youtube.com/live_chat?v=VIDEO_ID`, not a third-party page).
2. OBS → add **Browser** source → paste the URL.
3. Paste CSS into the source's **Custom CSS** box.

### Selectors (checked on a live popout chat, 2026-11-19)

| Element | Selector |
|---|---|
| Whole chat app | `yt-live-chat-app`, `yt-live-chat-renderer` |
| Message list / scroll area | `yt-live-chat-item-list-renderer`, `#item-scroller`, `#items` |
| Each normal message | `yt-live-chat-text-message-renderer` |
| Parts of a message | `#author-photo`, `#timestamp`, `#author-name`, `#chip-badges` / `#chat-badges`, `#message`, `#menu` (the ⋮ button) |
| Name + badges wrapper | `yt-live-chat-author-chip` |
| Owner / moderator / member / normal | attribute `author-type="owner"`, `"moderator"`, `"member"`, or empty `""` on the message renderer |
| Super Chat | `yt-live-chat-paid-message-renderer` |
| Super Sticker | `yt-live-chat-paid-sticker-renderer` |
| New-member message | `yt-live-chat-membership-item-renderer` |
| Top bar ("Top chat") | `yt-live-chat-header-renderer` |
| Pinned / banner message | `yt-live-chat-banner-manager` |
| Super Chat ticker row | `yt-live-chat-ticker-renderer` |
| Bottom bar ("Sign in to chat" / input) | `#input-panel`, `#panel-pages` |
| Welcome/"remember to be respectful" line | `yt-live-chat-viewer-engagement-message-renderer` |

> [!warning] Old guides are out of date
> The classes `.yt-live-chat-author-is-owner` / `.yt-live-chat-author-is-moderator` no longer exist. Use the `author-type` attribute. Also, OBS isn't signed in, so the bottom bar is "Sign in to chat" (inside `#input-panel`), not `yt-live-chat-message-input-renderer`.

### Starter CSS (tested)

Paste into the OBS browser source's Custom CSS box. Each rule needs `!important` to beat YouTube's own styles.

```css
/* 1. Transparent background */
html, body, yt-live-chat-app, yt-live-chat-renderer,
yt-live-chat-item-list-renderer, #item-scroller, #items, #item-offset,
yt-live-chat-text-message-renderer {
  background-color: transparent !important;
}

/* 2. Hide everything except the messages */
yt-live-chat-header-renderer,
yt-live-chat-ticker-renderer,
yt-live-chat-banner-manager,
yt-live-chat-viewer-engagement-message-renderer,
#input-panel, #panel-pages,
#author-photo, #menu,
yt-live-chat-text-message-renderer #timestamp {
  display: none !important;
}
#item-scroller { scrollbar-width: none !important; }
#item-scroller::-webkit-scrollbar { display: none !important; }

/* 3. Text */
yt-live-chat-text-message-renderer {
  font-family: "Segoe UI", sans-serif !important;
  font-size: 20px !important;
  padding: 4px 12px !important;
}
yt-live-chat-text-message-renderer #message {
  color: #fff !important;
  text-shadow: 0 1px 3px #000;   /* readable over gameplay */
}
yt-live-chat-text-message-renderer #author-name {
  font-weight: 700 !important;
  color: #9ad0ff !important;
}

/* 4. Role colors */
yt-live-chat-text-message-renderer[author-type="owner"]     #author-name { color: #ffd600 !important; }
yt-live-chat-text-message-renderer[author-type="moderator"] #author-name { color: #5da9ff !important; }
yt-live-chat-text-message-renderer[author-type="member"]    #author-name { color: #2ba640 !important; }

/* 5. Slide-in animation for new messages */
yt-live-chat-text-message-renderer { animation: slide-in .3s ease-out; }
@keyframes slide-in { from { opacity: 0; transform: translateX(-20px); } }
```

Optional extras:

```css
/* Each message as a rounded bubble */
yt-live-chat-text-message-renderer {
  background: rgba(0,0,0,.55) !important;
  border-radius: 10px !important;
  margin: 4px 8px !important;
}

/* Hide badges (moderator wrench, member badge, verified check) */
#chip-badges, #chat-badges, yt-live-chat-author-badge-renderer { display: none !important; }

/* Put the name on its own line above the message */
yt-live-chat-text-message-renderer yt-live-chat-author-chip { display: block !important; }
```

CSS can't match on text, so it can't hide or highlight one specific user (e.g. Nightbot) by name. That needs a tool from Option B.

Tips:
- To see your changes, right-click the source in OBS → **Refresh cache of current page**.
- To find a selector yourself, open the popout URL in Chrome, right-click a part of the chat → **Inspect**.
- The popout opens on **Top chat**, which filters messages. Right-click the source → **Interact** and switch to **Live chat** if you want every message (not tested whether OBS remembers this between restarts).
- Super Chats, stickers and new-member messages use their own renderers (table above) and weren't on screen during the test, so style them separately and check them on a real stream.

Known issues:
- YouTube renames elements now and then, which silently breaks presets.
- OBS 30.2.2 ignored custom CSS on YouTube chat; fixed in 30.2.3.
- Ready-made presets: [YouTube Live Chat CSS Presets (OBS forum)](https://obsproject.com/forum/resources/youtube-live-chat-css-presets.1182/), [OBS-ChatStyle](https://awesome.ecosyste.ms/projects/github.com%2Fvitorcarvalho67%2Fobs-chatstyle) (SCSS themes with CSS variables), [youtube-live-chat-overlay](https://github.com/EuSouGuil/youtube-live-chat-overlay).
- [YouTube Chat Overlay Lua script](https://obsproject.com/forum/resources/youtube-live-chat-overlay.2256/) adds the browser source for you, with a CSS field and transparent background (OBS 27+).

### Option B: chat-overlay tools (redraw the chat)

These read the chat and render their own HTML, so styling is stable and doesn't depend on YouTube's markup:

- **[Social Stream Ninja](https://socialstream.ninja/docs/guides)** (free): browser extension or desktop app captures popout chat, no API key. Gives a dashboard plus an overlay URL with options (dark mode, scale, fade/swipe effects, auto-hide), [templates](https://cache.socialstream.ninja/docs/templates), custom CSS and fonts. Merges Twitch/TikTok/etc. into one feed.
- **Streamlabs / StreamElements chat box widgets**: connect your YouTube account, pick a theme, edit colors/fonts, or switch to custom HTML/CSS/JS. (Not verified in this search; check their docs.)
- **Restream chat**, **StreamSpell** and similar multi-platform chat widgets.

## 3. Which to use

- Want one platform, free, full control and don't mind fixing CSS occasionally → Option A.
- Multistreaming, want animations/themes that don't break, or want to "feature" a message on screen → Social Stream Ninja.
- Already in Streamlabs/StreamElements for alerts → use their chat box widget so everything is in one place.

## Sources

- [Learn about Live Chat (YouTube Help)](https://support.google.com/youtube/answer/15268877)
- [Channel membership levels & perks (YouTube Help)](https://support.google.com/youtube/answer/7544492)
- [superChatEvents (YouTube Data API)](https://developers.google.com/youtube/v3/live/docs/superChatEvents)
- [Custom CSS for YouTube chat stopped working (OBS forum)](https://obsproject.com/forum/threads/custom-css-for-youtube-chat-stopped-working-suddenly.178495/)
- [Change style for a specific name (OBS forum)](https://obsproject.com/forum/threads/change-style-for-only-specific-name-in-youtube-chat-for-obs.167480)
- [YouTube Chat Tints userscript](https://greasyfork.org/scripts/457391-youtube-chat-tints/code) (viewer-side recoloring)
- [What are YouTube Super Chats? (vidIQ)](https://vidiq.com/blog/post/youtube-super-chat-guide/)
