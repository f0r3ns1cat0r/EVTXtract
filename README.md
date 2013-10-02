
Background
----------

EVTX records are XML fragments encoded using a Microsoft-specific binary XML representation.
Despite the convenient format, it is not easy to recover EVTX event log records from a corrupted file or unallocated space.
This is because the complete representation of a record often depends on other records found nearby.
The event log service recognizes similarities among records and refactors commonalities into "templates".
A template is a fixed structure with placeholders that reserve space for variable content.
The on-disk event log record structure is a reference to a template, and a list of substitutions (the variable content the replaces a placeholder in a template).
To decode a record into XML, the event log service resolves the template and replaces its placeholders with the entries of the substitution array.
Therefore, template corruption renders many records unrecoverable within the local 64KB "chunk".
However, the substitution array for the remaining records may still be intact.
If so, it may be possible to produce XML fragments that match the original records if the damaged template can be reconstructed.
For many common events, such as process creation or account logon, empirical testing demonstrates the relevant templates remain mostly constant.
In these cases, recovering event log records boils down to identifying appropriate templates found in other EVTX chunks.


Algorithm
---------

1. Scan for chunk signatures ("ElfChnk")
   - check header for sane values (0x80 <= size <= 0x200)
   - verify checksums (header, data)
2. Extract records from valid chunks found in (1)
3. Extract templates from valid chunks found in (1)
4. Scan for record signatures
   - check header for sane values
   - extract timestamp
   - attempt to parse substitutions
   - attempt to decode substitutions into EID, other fields
5. Reconstruct records by reusing old templates with recovered substitutions


TODO
----
1. DONE fix valid_record xml
2. DONE fix bug parsing chunks and records for valid records/templates
3. DONE ensure templates are not duplicated in the database
4. Figure out what happens with multiple runs on different files
     --> they must match, or the offsets will get screwed up
     --> therefore, the bug is to warn if the files differ, or if data already exists
5. DONE Logging issue
    Git/recover-evtx - [json-db\u25cf] \u00bb python extract_valid_evtx_records_and_templates.py tests/working-private/chunk_fragment/chunk_fragment.evtx
      No handlers could be found for logger "extract_records"
6. DONE Move argparser config to common area
7. figure out idempotency for state, related: #4
8. consider GZIPing the state and database files
9. write dumping scripts
10. DONE implement .get_template
11. DONE figure out where IDs should be used --> nowhere
12. DONE move things out of main
13. consider writing a Q&A/wizard style interface
14. develop flowchart describing usage
15. DONE for state/db file, on error, should the existing state be written out best-effort?
16. DONE Template.get_id is broken, only returns EID
17. DONE if project name specified, update the templatedb default
18. DONE templatedb filename should always end in _db.json
19. extract constraints in template DB for easy matching
20. add status output so the user knows that something happened. Use print() for this, not logging
21. DONE make substitution object/list things consistent. 2- or 3-tuples?
22. DONE nested template indices are incorrect. seems to be a string concat somewhere


JSON format
-----------
## State file
{
  version: int
  generator: str

  // metadata, for confirmation after the fact
  input_file: {
    size: int
    md5: str    // hex encoded md5sum of the first 0x100000 bytes of the input file
  }

  valid_chunk_offsets: [int, int, ...]
  potential_record_offsets: [int, int, ...]
  valid_records: [
    {
      offset: int
      eid: int
      xml: str
    },
    ...
  }
  lost_records: [
    {
      offset: int
      substitutions: [(str, str), (str, str), ...]
    },
    ...
  ]
  reconstructed_records: [
    {
      offset: int
      eid: int
      xml: str
    },
    ...
  ]
  unreconstructed_records: [
    {
      offset: int
      substitutions: [(str, str), (str, str), ...]
      reason: str
    },
    ...
  ]
}

## Template File
{
  version: int
  generator: str

  templates: {
    (int)eid: [
      {
        eid: int,
        id: str,
        xml: str,
      },
      ...
    ]
  }
}
