{{- /*
  Clean Markdown rendering of a single page for LLM/agent consumption.
  Served alongside the HTML version at the same URL with a .md suffix.
  Example: https://vouch.sh/docs/getting-started/  →  .../getting-started.md
*/ -}}
# {{ .Title }}

{{ with .Description }}> {{ . }}

{{ end -}}
Source: {{ .Permalink }}
{{ with .Lastmod }}Last updated: {{ .Format "2006-01-02" }}
{{ end }}
---

{{- partial "clean-md-export.html" .RawContent -}}
