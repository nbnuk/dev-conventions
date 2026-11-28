# Vocabularies (user-defined sets of values)

A **vocabulary** is a set of values that users define themselves, such as the
statuses on a local designation list ("Locally rare", "Possibly extinct") or,
later, tags. Each value is a **term**. Use these words in code. On screen, use
the owner's own word ("Statuses", "Tags"), never "vocabulary" or "term".

A fixed set that only developers change is an `enum`, not a vocabulary.

## Tenant vocabularies

Every vocabulary is a row in one tenant table, `vocab`, and its terms are rows
in `vocab_term`. Owners point at a vocabulary; the vocabulary does not record
its owner. One editor, picker, filter source and admin screen then serve every
vocabulary, and a vocabulary can later be shared by several owners (two lists
with the same statuses, one Stage vocabulary for all occurrences).

| Table | Domain | Columns |
|---|---|---|
| `vocab` | `Vocab` | `tenant_id`, `kind`, `name` (+ `id`, `version`, dates) |
| `vocab_term` | `VocabTerm` | `tenant_id`, `vocab_id`, `label`, `description`, `sort_order` (+ `id`, `version`, dates) |

- `kind` is a `VocabKind` enum value (`SPECIES_LIST_DESIGNATION_STATUS`, …) saying what the
  vocabulary is for, so a screen offers only vocabularies of its kind. Add a
  kind when a new kind of owner gets user-defined values.
- `name` is what a person sees when choosing or managing vocabularies. A
  vocabulary created for one owner takes the owner's name and is renamed with it.
- `label` is required, at most 255 characters, and unique per vocabulary
  ignoring case (a unique index on `(vocab_id, lower(label))`, which also serves
  "the vocabulary's terms").
- `description` is optional text.
- `sort_order` is the order the user gave the terms. It starts as the order they
  were added; screens list terms, and sort by term, in this order.
- Both tables carry `tenant_id` and are `MultiTenant`.

### Owners and records

| Piece | Rule | Example |
|---|---|---|
| Owner → vocabulary | property `vocab`, column `vocab_id` (null when the owner has none) | `TenantSpeciesList.vocab` |
| Owner's terms, in order | a transient `getTerms()` reading `vocab.terms` | `TenantSpeciesList.terms` |
| Record with one term | property `term`, column `vocab_term_id`, indexed | `TenantSpeciesListItem.term` |
| Record with several single-term fields | property `<field>Term`, column `<field>_vocab_term_id` | `Occurrence.stageTerm` (later) |
| Record with several terms (tags) | a join table (record id, `vocab_term_id`), indexed on the term | (later) |

The service creates the owner's vocabulary with the owner, so an owner that
has values always has one.

### Rules

- **Records point at the term's id**, never copy its label. Renaming a term
  changes it everywhere at once.
- **A term in use cannot be deleted.** Screens show how many records use each
  term and don't offer delete for those in use; the service refuses it, and the
  record's foreign key to `vocab_term` refuses it in the database. Merging terms
  is a separate feature when someone needs it.
- **The service keeps a record's term within its owner's vocabulary.** It looks
  the term up by the owner's `vocab_id` and id, never by id alone. The
  database's foreign key only checks that the term exists, so the service is
  the only writer; a bulk import in SQL runs a check query afterwards.
- **Changing terms is changing the owner.** Term edits touch the owner's
  `lastUpdated`, so the owner's version moves and an edit form opened earlier
  is refused (see [`edit-form-versions.md`](edit-form-versions.md)).
- **Filters** list a vocabulary's terms from `vocab_term` and count records by
  the indexed term column or join table, the same way for every owner.

## Managing terms on screen

Use the components plugin's **term editor** (`<components:termeditor/>`) in the
owner's edit form. It is a form field, not a separate save: it submits the
whole set as `terms[0].id`, `terms[0].label`, `terms[0].description`, … in the
order shown, and the owner's service saves the set with the owner.

- The command binds the rows into `List<TermEditorRow> terms` and validates them
  with `TermEditorRow.validate(...)` (blank, too long, duplicate labels).
- The service compares the submitted rows with the stored terms: rows with an id
  are updated, rows without one are created, stored terms missing from the
  submission are deleted (refused when in use). `sort_order` is the row's
  position.
- The host passes each term's usage count and the words to use ("status",
  "species").

## Choosing a term as you act on a record

When a screen acts on many rows and each needs one term (adding species to a
designation list with a status), use the components plugin's **single term
picker**. It is server-rendered and HTMX-driven; its script only shows, places
and closes the popup.

- `<components:singletermpicker id=…/>` once on the page: the shared popup.
- `<components:singletermpickerAction …/>` on each row: "Add as {term}" with the
  default term, and a toggle that GETs the row's choices into the popup.
- The host's options endpoint renders `<components:singletermpickerOptions …/>`
  with the vocabulary as it is now, so a term created from one row is offered
  on the next.
- A pick POSTs the row's values with `termId`, or `termLabel` for a typed term.
  The host finds a typed label ignoring case, else creates it in the same
  transaction as the record.
- The host remembers the default term (the last one picked, e.g. in the
  session). When a pick changes it, the response appends
  `<components:singletermpickerDefault …/>`, an out-of-band swap that redraws
  every row's main button.

Changing one record's term in a table can stay a plain `<select>` of the terms.
Choosing several terms (tags) will be a separate multi term picker rather than
an option on this one, even if some markup is repeated, to keep each simple.
