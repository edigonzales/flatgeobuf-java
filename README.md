# flatgeobuf-java

Bibliothek zum Exportieren von Geodaten aus JDBC-Tabellen in FlatGeobuf- **und** Parquet-Dateien (inklusive Spatial Index für FlatGeobuf) mit Unterstützung für GeoPackage-Geometry-Blobs.

## Umsetzung

- **Separation of Concern**: Die Export-Logik arbeitet mit generischen Tabellen-Deskriptoren (`TableDescriptorProvider`) und einem austauschbaren `GeometryReader` im neutralen Paket `ch.so.agi.cloudformats`. Dadurch ist der Export nicht an GeoPackage gebunden.
- **GeoPackage**: `GeoPackageGeometryReader` extrahiert WKB aus GPKG-Geometry-Blobs (Magic-Bytes, Flags, Envelope) und konvertiert sie nach JTS.
- **ili2db-Tabellen**: `Ili2dbTableDescriptorProvider` nutzt die vorgegebene SQL-Abfrage und liefert Tabellenname, Geometriespalte, SRID und GeometryType.
- **FlatGeobuf**: `FlatGeobufTableWriter` erstellt Header/Features und schreibt einen Hilbert-sortierten `PackedRTree` Index für effiziente Streaming- und Range-Requests.
- **Parquet**: `ParquetTableWriter` schreibt Parquet-Dateien mit Geometry/Geography Logical Types (ab Parquet 1.17.0) und unterstützt konfigurierbare Row Group Sizes.

## Verwendung

### Export aus GeoPackage (ili2db-Layout) nach FlatGeobuf

```java
try (Connection connection = DriverManager.getConnection("jdbc:sqlite:your.gpkg")) {
    FlatGeobufExporter exporter = new FlatGeobufExporter(new GeoPackageGeometryReader());
    exporter.exportTables(connection, new Ili2dbTableDescriptorProvider(), Path.of("output"));
}
```

- Erzeugt pro Tabelle eine `<tablename>.fgb` Datei.
- Tabellen werden per SQL-Abfrage aus `T_ILI2DB_TABLE_PROP` selektiert.
- Spatial Index ist standardmäßig aktiv (Node Size 16).

### Export aus beliebigen JDBC-Tabellen (WKB in BLOB) nach FlatGeobuf

```java
FlatGeobufTableWriter writer = new FlatGeobufTableWriter(new WkbGeometryReader());
TableDescriptor table = new TableDescriptor("my_table", "geom", 4326, (byte) GeometryType.MultiPolygon);
try (OutputStream out = Files.newOutputStream(Path.of("my_table.fgb"))) {
    writer.writeTable(connection, table, out);
}
```

### Export nach Parquet

```java
try (Connection connection = DriverManager.getConnection("jdbc:sqlite:your.db")) {
    ParquetExporter exporter = new ParquetExporter(new WkbGeometryReader());
    exporter.exportTables(connection, tableDescriptorProvider, Path.of("output"));
}
```

- Erzeugt pro Tabelle eine `<tablename>.parquet` Datei.
- Geometry/Geography Logical Types werden im Parquet-Schema gesetzt (WKB-Encoding).
- Die Row Group Size kann über `ParquetWriteOptions.builder().rowGroupSize(...)` konfiguriert werden.

### Export aus beliebigen JDBC-Tabellen nach Parquet (direkter Writer)

```java
ParquetTableWriter writer = new ParquetTableWriter(new WkbGeometryReader());
TableDescriptor table = new TableDescriptor("my_table", "geom", 4326, (byte) GeometryType.MultiPolygon);
Path target = Path.of("my_table.parquet");
ParquetTableWriter.ParquetWriteOptions options = ParquetTableWriter.ParquetWriteOptions.builder()
        .rowGroupSize(128 * 1024 * 1024)
        .build();
writer.writeTable(connection, table, target, options);
```

### Hinweise für Streaming/HTTP Range Requests

- Die erzeugten FlatGeobuf-Dateien enthalten einen Spatial Index (`PackedRTree`).
- Der Index erlaubt effiziente Abfragen per Byte-Range (z.B. HTTP Range Requests).
