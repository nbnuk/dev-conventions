# Vocabularies (user-defined sets of values)

A **vocabulary** is a set of values that users define themselves, such as the
statuses on a local designation list ("Locally rare", "Possibly extinct") or,
later, tags. Each value is a **term**. Use these words in code. On screen, use
the owner's own word ("Statuses", "Tags"), never "vocabulary" or "term".

A fixed set that only developers change is an `enum`, not a vocabulary.

## Owned vocabularies

A vocabulary belongs to one **owner** record and is created, edited and
deleted with it. There is no shared global vocabulary table: each owner has its
own term table, and every owner follows the same names and shape. A shared
vocabulary table pays off only when several owners share one set of values;
add one then, not before.

### Names

| Piece | Rule | Example |
|---|---|---|
| Term domain class | `<Owner>Term` | `TenantSpeciesListTerm` |
| Table | `<owner_table>_term` | `tenant_species_list_term` |
| Owner's terms | `terms` | `TenantSpeciesList.terms` |
| Record that uses a term | property `term`, column `term_id` | `TenantSpeciesListItem.term` |

### Columns

- `label` — required, at most 255 characters, unique per owner ignoring case
  (a unique index on `(<owner>_id, lower(label))`).
- `description` — optional text.
- `sort_order` — the order the user gave the terms. It starts as the order they
  were added; screens list terms, and sort by term, in this order.
- Tenant-owned terms carry `tenant_id` and are `MultiTenant`, like their owner.

### Rules

- **Records point at the term's id**, never copy its label. Renaming a term
  changes it everywhere at once.
- **A term in use cannot be deleted.** Screens show how many records use each
  term and don't offer delete for those in use; the service refuses it as a
  backstop. Merging terms is a separate feature when someone needs it.
- **The database checks a term belongs to the same owner** as the record using
  it: a unique constraint on the term table's `(<owner>_id, id)` and a
  composite foreign key from the record's `(<owner>_id, term_id)`.
- **Changing terms is changing the owner.** Term edits touch the owner's
  `lastUpdated`, so the owner's version moves and an edit form opened earlier
  is refused (see [`edit-form-versions.md`](edit-form-versions.md)).

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

Choosing a term for a record (a dropdown, or a picker that can add a term on
the spot) belongs to the owning screen for now. Extract a shared term picker
when a second vocabulary needs one.
