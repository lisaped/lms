# Flagstaff Group LMS — WordPress Plugin Build Notes

**Date:** March 23, 2026
**Plugin Version:** 1.0.0
**Requires:** WordPress 6.0+, PHP 8.0+
**License:** GPL-2.0-or-later

---

## Project Summary

A full-featured, branded Learning Management System (LMS) WordPress plugin for Flagstaff Group, comparable in scope to LearnDash. Built from scratch following WordPress Plugin Handbook conventions and security best practices.

---

## Requirements Gathered

| Dimension | Choice |
|---|---|
| Core Features | Courses & Lessons, Student Progress Tracking, Certificates, Enrollments & Access Control |
| Access Model | Membership / Subscription |
| Brand Color | Red (`#C8102E`) |

---

## Plugin File Structure

```
flagstaff-group-lms/
├── flagstaff-group-lms.php          # Main bootstrap, plugin header, lifecycle hooks
├── uninstall.php                    # Safe cleanup on plugin deletion
├── readme.txt                       # WordPress.org-compatible readme
│
├── includes/
│   ├── class-fg-lms-plugin.php      # Singleton core, wires all subsystems
│   ├── class-fg-lms-installer.php   # DB schema creation, activation, upgrades
│   ├── class-fg-lms-cpt.php         # Custom Post Types & Taxonomies
│   ├── class-fg-lms-enrollment.php  # Enrollment CRUD & queries
│   ├── class-fg-lms-membership.php  # Plans, subscriptions, access control
│   ├── class-fg-lms-progress.php    # Per-lesson tracking, course % completion
│   ├── class-fg-lms-certificate.php # Certificate issuance & download endpoint
│   └── class-fg-lms-shortcodes.php  # [fg_lms_catalog], [fg_lms_dashboard], etc.
│
├── admin/
│   ├── class-fg-lms-admin.php       # Menus, asset enqueue, meta boxes, POST handlers
│   └── views/
│       ├── dashboard.php            # KPI stats + recent enrollments
│       ├── courses.php              # Course list with lesson/student counts
│       ├── students.php             # Student manager + manual enroll/unenroll
│       ├── memberships.php          # Plan CRUD (list + form)
│       ├── reports.php              # Per-course progress overview
│       ├── settings.php             # Branding, pages, certificate config
│       ├── meta-course-builder.php  # Drag-and-drop lesson ordering meta box
│       ├── meta-course-settings.php # Free flag, passing score
│       └── meta-lesson-settings.php # Parent course assignment
│
├── public/
│   ├── class-fg-lms-public.php      # Asset enqueue, form handling, content gating
│   └── views/
│       ├── course-catalog.php       # Grid catalog (shortcode + archive template)
│       ├── course-single.php        # Single course with hero, curriculum sidebar
│       ├── lesson-single.php        # Lesson player with nav, mark-complete
│       └── student-dashboard.php    # Enrolled courses, progress, certificates
│
├── templates/
│   └── certificate-template.php    # Printable HTML certificate page
│
└── assets/
    ├── css/admin.css                # Admin UI styles (stats grid, tables, sortable)
    ├── css/public.css               # Frontend styles (cards, progress bars, sidebar)
    ├── js/admin.js                  # jQuery UI sortable for course builder
    └── js/public.js                 # AJAX mark-complete, progress animation
```

---

## Database Schema (6 custom tables)

### `{prefix}fg_lms_plans`
Membership plan definitions.

| Column | Type | Notes |
|---|---|---|
| id | BIGINT UNSIGNED | PK |
| name | VARCHAR(200) | |
| description | TEXT | |
| price | DECIMAL(10,2) | |
| duration_days | INT UNSIGNED | 0 = lifetime |
| course_access | ENUM('all','specific') | |
| status | ENUM('active','inactive') | |
| created_at | DATETIME | |

### `{prefix}fg_lms_memberships`
User subscriptions to plans.

| Column | Type | Notes |
|---|---|---|
| id | BIGINT UNSIGNED | PK |
| user_id | BIGINT UNSIGNED | |
| plan_id | BIGINT UNSIGNED | FK → plans |
| status | ENUM('active','expired','cancelled') | |
| started_at | DATETIME | |
| expires_at | DATETIME | NULL = lifetime |

### `{prefix}fg_lms_plan_courses`
Maps specific courses to plans when `course_access = 'specific'`.

### `{prefix}fg_lms_enrollments`
Per-user, per-course enrollment records.

| Column | Type | Notes |
|---|---|---|
| id | BIGINT UNSIGNED | PK |
| user_id | BIGINT UNSIGNED | UNIQUE with course_id |
| course_id | BIGINT UNSIGNED | |
| status | ENUM('active','completed','expired') | |
| enrolled_at | DATETIME | |
| completed_at | DATETIME | NULL until done |

### `{prefix}fg_lms_progress`
Per-user, per-lesson/quiz completion.

| Column | Type | Notes |
|---|---|---|
| id | BIGINT UNSIGNED | PK |
| user_id | BIGINT UNSIGNED | UNIQUE with lesson_id |
| course_id | BIGINT UNSIGNED | |
| lesson_id | BIGINT UNSIGNED | fg_lesson or fg_quiz post ID |
| status | ENUM('not_started','in_progress','completed') | |
| score | TINYINT UNSIGNED | 0–100, NULL for lessons |
| completed_at | DATETIME | |

### `{prefix}fg_lms_certificates`
Issued certificates (one per user per course).

| Column | Type | Notes |
|---|---|---|
| id | BIGINT UNSIGNED | PK |
| user_id | BIGINT UNSIGNED | UNIQUE with course_id |
| course_id | BIGINT UNSIGNED | |
| certificate_number | VARCHAR(64) | Unique, URL-safe |
| issued_at | DATETIME | |

---

## Custom Post Types & Taxonomies

| Post Type | Slug | Public | Notes |
|---|---|---|---|
| `fg_course` | `/courses/` | Yes | Archive enabled |
| `fg_lesson` | `/lessons/` | Yes | Assigned to course via post meta |
| `fg_quiz` | `/quizzes/` | Yes | Same template as lesson |
| `fg_question` | — | No | Child of fg_quiz |

| Taxonomy | Object | Hierarchical |
|---|---|---|
| `fg_course_cat` | fg_course | Yes |
| `fg_course_tag` | fg_course | No |

---

## Shortcodes

| Shortcode | Attributes | Description |
|---|---|---|
| `[fg_lms_catalog]` | `columns`, `per_page`, `category` | Course grid catalog |
| `[fg_lms_dashboard]` | — | Student personal dashboard |
| `[fg_lms_course_progress]` | `course_id` | Progress bar for a course |
| `[fg_lms_enroll_button]` | `course_id` | Smart CTA (Enroll / Continue / Login / Plans) |

---

## Plugin Options (Settings API)

| Option Key | Default | Description |
|---|---|---|
| `fg_lms_brand_name` | Flagstaff Group LMS | Platform display name |
| `fg_lms_primary_color` | `#C8102E` | Brand red — drives CSS custom property |
| `fg_lms_secondary_color` | `#1A1A1A` | Secondary/text color |
| `fg_lms_catalog_page_id` | 0 | Page with `[fg_lms_catalog]` |
| `fg_lms_dashboard_page_id` | 0 | Page with `[fg_lms_dashboard]` |
| `fg_lms_certificate_logo_url` | — | Logo shown on certificates |
| `fg_lms_cert_signatory_name` | — | Printed signature name |
| `fg_lms_cert_signatory_title` | — | Printed signature title |

---

## Key Hooks & Filters

| Hook | Type | Description |
|---|---|---|
| `fg_lms_user_subscribed` | action | Fires after a user subscribes to a plan |
| `fg_lms_user_enrolled` | action | Fires after a user is enrolled in a course |
| `fg_lms_user_unenrolled` | action | Fires after unenrollment |
| `fg_lms_lesson_completed` | action | Fires after a lesson is marked complete |
| `fg_lms_course_completed` | action | Fires when course progress reaches 100% |
| `fg_lms_certificate_issued` | action | Fires after a certificate is auto-issued |

---

## Security Measures Applied

- All admin pages guarded by `current_user_can('manage_options')` capability checks
- All form submissions use WordPress nonces (`wp_verify_nonce`)
- All DB queries use `$wpdb->prepare()` — no string-interpolated SQL
- All input sanitized with `sanitize_text_field`, `absint`, `wp_kses_post`, etc.
- All output escaped with `esc_html`, `esc_attr`, `esc_url` at render time
- `wp_unslash()` applied before sanitization on `$_POST` / `$_GET` values
- Uninstall guarded by `WP_UNINSTALL_PLUGIN` constant check

---

## Installation

1. Upload `flagstaff-group-lms.zip` via **Plugins → Add New → Upload Plugin**
2. Activate the plugin
3. Go to **Flagstaff LMS → Settings** — set brand name, colors, and page IDs
4. Create a **Course Catalog** page containing `[fg_lms_catalog]`
5. Create a **Student Dashboard** page containing `[fg_lms_dashboard]`
6. Create your first course via **Flagstaff LMS → Courses → New Course**
7. Add lessons via **Add New Lesson**, assign them to the course in the sidebar meta box
8. Drag-and-drop lesson order inside the **Course Builder** meta box on the course edit screen
9. Create a membership plan via **Flagstaff LMS → Memberships → New Plan**

---

## Deliverables

| File | Description |
|---|---|
| `flagstaff-group-lms.zip` | WordPress-installable plugin package (53 KB) |
| `flagstaff-group-lms/` | Full source directory (31 files) |
