```cypher
:param {
  // Define the file path root and the individual file names required for loading.
  // https://neo4j.com/docs/operations-manual/current/configuration/file-locations/
  file_path_root: 'file:///', // Change this to the folder your script can access the files at.
  file_0: 'games.csv'
};

// CONSTRAINT creation
// -------------------
//
// Create node uniqueness constraints, ensuring no duplicates for the given node label and ID property exist in the database. This also ensures no duplicates are introduced in future.
//
// NOTE: The following constraint creation syntax is generated based on the current connected database version 5.27-aura.
CREATE CONSTRAINT `gameName_juegos_uniq` IF NOT EXISTS
FOR (n: `juegos`)
REQUIRE (n.`gameName`) IS UNIQUE;
CREATE CONSTRAINT `season_temporadas_uniq` IF NOT EXISTS
FOR (n: `temporadas`)
REQUIRE (n.`season`) IS UNIQUE;
CREATE CONSTRAINT `playerId_jugadores_uniq` IF NOT EXISTS
FOR (n: `jugadores`)
REQUIRE (n.`playerId`) IS UNIQUE;

:param {
  idsToSkip: []
};

// NODE load
// ---------
//
// Load nodes in batches, one node label at a time. Nodes will be created using a MERGE statement to ensure a node with the same label and ID property remains unique. Pre-existing nodes found by a MERGE statement will have their other properties set to the latest values encountered in a load file.
//
// NOTE: Any nodes with IDs in the 'idsToSkip' list parameter will not be loaded.
LOAD CSV WITH HEADERS FROM ($file_path_root + $file_0) AS row
WITH row
WHERE NOT row.`gameName` IN $idsToSkip AND NOT row.`gameName` IS NULL
CALL {
  WITH row
  MERGE (n: `juegos` { `gameName`: row.`gameName` })
  SET n.`gameName` = row.`gameName`
} IN TRANSACTIONS OF 10000 ROWS;

LOAD CSV WITH HEADERS FROM ($file_path_root + $file_0) AS row
WITH row
WHERE NOT row.`season` IN $idsToSkip AND NOT toInteger(trim(row.`season`)) IS NULL
CALL {
  WITH row
  MERGE (n: `temporadas` { `season`: toInteger(trim(row.`season`)) })
  SET n.`season` = toInteger(trim(row.`season`))
} IN TRANSACTIONS OF 10000 ROWS;

LOAD CSV WITH HEADERS FROM ($file_path_root + $file_0) AS row
WITH row
WHERE NOT row.`playerId` IN $idsToSkip AND NOT row.`playerId` IS NULL
CALL {
  WITH row
  MERGE (n: `jugadores` { `playerId`: row.`playerId` })
  SET n.`playerId` = row.`playerId`
} IN TRANSACTIONS OF 10000 ROWS;


// RELATIONSHIP load
// -----------------
//
// Load relationships in batches, one relationship type at a time. Relationships are created using a MERGE statement, meaning only one relationship of a given type will ever be created between a pair of nodes.
LOAD CSV WITH HEADERS FROM ($file_path_root + $file_0) AS row
WITH row 
CALL {
  WITH row
  MATCH (source: `temporadas` { `season`: toInteger(trim(row.`season`)) })
  MATCH (target: `juegos` { `gameName`: row.`gameName` })
  MERGE (source)-[r: `se_tienen`]->(target)
  SET r.`gameName` = row.`gameName`
} IN TRANSACTIONS OF 10000 ROWS;

LOAD CSV WITH HEADERS FROM ($file_path_root + $file_0) AS row
WITH row 
CALL {
  WITH row
  MATCH (source: `jugadores` { `playerId`: row.`playerId` })
  MATCH (target: `temporadas` { `season`: toInteger(trim(row.`season`)) })
  MERGE (source)-[r: `participan`]->(target)
  SET r.`season` = toInteger(trim(row.`season`))
} IN TRANSACTIONS OF 10000 ROWS;
