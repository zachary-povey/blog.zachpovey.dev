{{- $postTitle := replace .File.ContentBaseName "_" " " | title -}}
+++
title = '{{ $postTitle }}'
date = {{ .Date }}
draft = true
pinned: false
+++