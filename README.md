# SWYNEX Project Architecture — Public Distribution Service

This starter repository documents and organizes a React + Express + MySQL Public Distribution Service project.

## Contents
- `ARCHITECTURE.md` — product idea, system design, data models, REST API, screens, flows, and security notes.
- `app/` — existing frontend/backend source files copied from the supplied project archive.

## Local setup
1. Install Node.js LTS and MySQL.
2. Create the database and tables expected by `app/Backend/config/db.js` and the models.
3. Create a local `.env` file in `app/Backend/` (never commit it). Set the DB connection variables and JWT secret required by the project.
4. In `app/Backend/`, run `npm install`, then `npm run dev`.
5. In `app/Frontend/`, run `npm install`, then `npm start`.

Check the existing configuration and SQL schema before running; this archive does not include a production-ready migration/seed setup.

## Important
- Do not commit `.env`, credentials, personal data, `node_modules`, or debug logs.
- Review authentication and authorization before deploying. The architecture document lists key security improvements.
