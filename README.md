# Qur'an Website Plan (FastAPI + React + Tailwind + Postgres)

A plan for building a Qur'an website with text and audio, using existing verified sources for the Qur'anic text. The Arabic text must never be typed or retyped by hand.

---

## 1. Text sources

- **Tanzil.net**: downloadable Uthmani and simple Arabic text (XML, SQL, plain text) plus many translations. The license allows use if the text is kept unaltered and Tanzil is credited. They publish checksums so you can verify the text has not been corrupted.
- **Quran.com / Quran Foundation API**: text, translations, word-by-word data, tafsir and audio metadata. Check their current docs, since access has moved toward registered API clients.
- **AlQuran Cloud API**: a simple free API with text, translations and audio editions. Good for prototyping.

## 2. Audio sources

- **EveryAyah.com**: per-verse MP3s from many reciters with predictable URLs.
- **Quran.com audio CDN**: per-verse and per-surah audio with timing data for verse highlighting.
- Check each reciter's or host's license before shipping.
- Hotlinking someone else's CDN is unreliable. For production, host the files yourself (S3, Cloudflare R2) if the license allows it.

---

## 3. Architecture

### Database (Postgres)

Tables:

- `surahs`: id, name_arabic, name_english, revelation_place, ayah_count
- `ayahs`: id, surah_id, number, `text_uthmani`, `text_search` (diacritics stripped, used for search only)
- `translations`: id, ayah_id, language, translator, text
- `reciters`: id, name, style, base_url
- `audio_files`: id, ayah_id, reciter_id, url, optional timing data

Extensions: `pg_trgm` and `unaccent` for search.

### Backend (FastAPI)

- SQLAlchemy 2.0 (async) with asyncpg
- Alembic for migrations
- Pydantic models for responses

Endpoints:

- `GET /surahs`
- `GET /surahs/{id}/ayahs`
- `GET /ayahs/{surah}/{ayah}`
- `GET /search?q=`
- `GET /reciters`

Notes:

- The data never changes, so add `Cache-Control` and `ETag` headers.
- You could also export the whole text as static JSON for the frontend.
- Write the Tanzil import as a standalone script run once, then compare the stored text against Tanzil's published checksum.

### Frontend (React + Tailwind)

- Vite for the build
- React Router for routes such as `/surah/:id`
- TanStack Query for fetching and caching
- Zustand for global audio state (current ayah, reciter, speed, repeat) so the player survives navigation

Tailwind and RTL:

- Use logical utilities (`ms-`, `me-`, `ps-`, `pe-`, `text-start`) instead of left/right ones.
- Set `dir="rtl"` on Arabic blocks.
- Load an Arabic font with `@font-face`: Amiri Quran, KFGQPC Uthmanic Hafs or Scheherazade New.
- Use a generous line height so diacritics do not clip.

Audio player:

- Persistent player with surah and ayah playback
- Repeat, speed control and reciter selection
- Highlight the current ayah by listening to the audio element's `timeupdate` event and comparing against per-ayah timing data

---

## 4. Build order

1. Database models, migration and Tanzil import. Verify the text against the checksum.
2. API endpoints: surahs, ayahs, reciters, search.
3. Reading view with Arabic text and one translation.
4. Audio player with per-ayah playback and highlighting.
5. Search, bookmarks and last-read position.
6. Later: tafsir, juz/hizb navigation, offline support (PWA).

---

## 5. Things to get right early

- **Never modify the Arabic text**, including diacritics and ayah markers. Run a checksum or diff against the source after import.
- **Normalize for search only.** Keep a separate diacritic-free column for searching and always display the original.
- **Use Unicode text, not images**, for accessibility and searchability. Test rendering on mobile, since poor fonts can break tajweed marks.
- **Credit your sources** (Tanzil, translators, audio providers) on the site.
- **Check licenses** for text, translations and audio before going live.

---

## 6. Suggested project structure

```
quran-app/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── routers/
│   │   │   ├── surahs.py
│   │   │   ├── ayahs.py
│   │   │   ├── search.py
│   │   │   └── reciters.py
│   │   └── db.py
│   ├── scripts/
│   │   └── import_tanzil.py
│   ├── alembic/
│   └── requirements.txt
└── frontend/
    ├── src/
    │   ├── components/   (AyahList, AudioPlayer, SearchBar)
    │   ├── pages/        (Home, Surah, Search, Bookmarks)
    │   ├── store/        (audio store)
    │   └── api/
    ├── tailwind.config.js
    └── vite.config.ts
```

---

## 7. Next steps

- Scaffold the backend: models, migration, Tanzil import script and endpoints.
- Then build the React reading view, followed by the audio player.