+++
title = "Collections"
sort_by = "date"
template = "prose.html"
page_template = "prose.html"

[extra]
toc = true
lang = "en"
date_format = "%b %-d, %Y"
back_to_top = true
comment = false
copy = true
outdate_alert = false
outdate_alert_days = 120
outdate_alert_text_before = "This note was last updated "
outdate_alert_text_after = " days ago and may be out of date."
+++

## Projects

{{ collection(file="projects.toml") }}

## Experiences

{{ collection(file="experiences.toml") }}

## Bookmarks

{{ collection(file="bookmarks.toml") }}
