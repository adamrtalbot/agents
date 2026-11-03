# Tool citations from module metadata

Add dynamic citations separately from the helper migration. This example targets nf-core-utils 0.7.0. Reference implementations: [seqinspector#241](https://github.com/nf-core/seqinspector/pull/241) and [createpanelrefs#107](https://github.com/nf-core/createpanelrefs/pull/107).

## Collect citations

The plugin matches tool names from version tuples against the `tools:` keys in module `meta.yml` files. Publication metadata supplies author/year citations and bibliography entries, with DOI, homepage and description fallbacks. Missing metadata can produce `No citation metadata found` warnings; absence of warnings does not prove completeness.

The example assumes `ch_upstream_versions` contains version tuples from upstream tasks only. Do not collect the global `versions` topic if MultiQC consumes the resulting description and emits to that topic: collection waits for MultiQC, which waits for collection. Use an upstream-only channel or reroute downstream version emissions. Adding extra tools or filtering tuples after reading the topic does not break that dependency cycle.

```groovy
include { citationsOnTheFly; methodsDescriptionText } from 'plugin/nf-core-utils'

def methods_description = ch_upstream_versions
    .toList()
    .map { versions ->
        methodsDescriptionText(
            multiqc_custom_methods_description.toString(),
            citationsOnTheFly(
                versions,
                files("${projectDir}/modules/**/meta.yml").collect { path -> path.toString() },
                ['multiqc'],
            ),
        )
    }
```

List every tool that runs after the collection, including MultiQC plugins, in the extra-tools argument. Keep pipeline-specific template substitutions in the third `methodsDescriptionText` argument as a map. `.toList()` preserves tuples and emits a list even when the upstream channel is empty.

## Supply verified metadata

Compare emitted tool names with `tools:` keys before testing; 0.7.0 also matches case-insensitively. Agree the metadata location with the user: upstream nf-core/modules changes or local module patches. A local overlay is an alternative only after verifying that the pinned plugin accepts it and resolves duplicates as intended.

Resolve DOIs and copy author, year, title and journal fields from Crossref or PubMed. Do not invent bibliographic details or replace an existing cited paper without approval.

```yaml
tools:
  - mosdepth:
      doi: "10.1093/bioinformatics/btx699"
      publication:
        author: "Pedersen B.S. & Quinlan A.R."
        year: 2018
        title: "Mosdepth: quick coverage calculation for genomes and exomes"
        source: "Bioinformatics"
```

Bulk edits must handle both quoted and unquoted tool keys. Compare the changed modules with the intended list.

## Patch and verify

- For local metadata changes, run `nf-core modules patch <module>`. Inspect the resulting patch, including gitignored test files, for unrelated drift. Compare unexpected changes against the pinned `git_sha` in `modules.json`; do not overwrite existing local work to make the patch smaller.
- Run the affected nf-tests and inspect the generated `methods_description_mqc.yaml` and rendered MultiQC report. Confirm that tools passed as extras appear and bibliography entries contain the verified metadata. Snapshots alone may not cover the generated text.
- Resolve missing-citation warnings for scientific tools. If the pipeline ignores warnings for generic shell utilities, ignore each exact message separately in its existing snapshot filters, never the blanket `No citation metadata found` pattern. An ignored warning does not supply a citation.
- Update the pipeline's `CITATIONS.md` for newly attributed tools using the verified metadata.
- Regenerate affected snapshots through nf-test after reviewing the changes. Document the metadata location, patch maintenance and any approved paper replacements in the pipeline's contribution documentation and PR.
