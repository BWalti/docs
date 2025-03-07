---
title: SQL Grammar
summary: The full SQL grammar for CockroachDB, generated automatically from the CockroachDB code.
toc: true
back_to_top: true
docs_area: reference.sql
---

{% assign rd = site.data.releases | where_exp: "rd", "rd.major_version == page.version.version" | first %}

{% if rd %}
{% assign remote_include_version = page.version.version | replace: "v", "" %}
{% else %}
{% assign remote_include_version = site.versions["stable"] | replace: "v", "" %}
{% endif %}

<style>
/* TODO(mjibson): reduce to header height once it no longer changes on scroll */
a[name]::before {
	content: '';
	display: block;
	height: 80px;
	margin: -80px 0 0;
}
a[name]:focus {
	outline: 0;
}
</style>

{{site.data.alerts.callout_success}}
This page describes the full CockroachDB SQL grammar. However, as a starting point, it's best to reference our [SQL statements pages](sql-statements.html) first, which provide detailed explanations and examples.
{{site.data.alerts.end}}

{% comment %}
TODO: clean up the SQL diagrams not to link to these missing nonterminals.
{% endcomment %}
<a id="col_label"></a>
<a id="column_constraints"></a>
<a id="column_name"></a>
<a id="count"></a>
<a id="fk_column_name"></a>
<a id="limit_val"></a>
<a id="offset_val"></a>
<a id="ref_column_name"></a>
<a id="simple_"></a>
<a id="table_alias_name"></a>
<a id="target_name"></a>
<a id="timestamp"></a>

<div>
{% remote_include https://raw.githubusercontent.com/cockroachdb/generated-diagrams/release-{{ remote_include_version }}/grammar_svg/stmt_block.html %}
</div>
