# IMROI 4.0.0. LTR datamodel implementatie 
1e concept opzet verstrekt op 20-10-2020
2e definitief verstrekt op 14-1-2020

## Inleiding

Dit document is bedoeld voor IMROI 4.0.0. LTR. Dit is een samenvoeging van datamodel 3.0.4. en 3.2.0. Het uitgangspunt was de modellen samen te voegen zodat diverse applicaties compatible zijn met het datamodel. De samenvoeging is een startpunt voor de komende periode en is gebaseerd op het document &quot;IMROI 4.0.0. samengevoegd datamodel&quot;.

## Uitwerking datamodel

**Changes**

- Tabel objecten.aanwezig kolom bijzonderheid – uit 3.2.0. 
- Tabel objecten.bouwlagen kolom fotografie_id integer – uit 3.0.4.
- Constraint bouwlagen_fotografie_id_fk – uit 3.0.4.
- Tabel objecten.afw_binnendekking verwijst naar objecten.afw_binnendekking_type voor soort
- Tabel objecten.object_status_type – vervanger van historie_status_type
- Tabel objecten.object_matrix_code – vervanger van historie_matrix_code
- Tabel objecten.object_aanpassing_type – vervanger van objecten.historie_aanpassing_type 
- Tabel objecten.contactpersoon verwijst naar objecten.contactpersoon_type voor soort
- Tabel objecten.dreiging kolom omschrijving – uit 3.2.0.
- Tabel objecten.label verwijst naar objecten.label_type voor soort
- Tabel objecten.opstelplaats verwijst naar objecten.opstelplaats_type voor soort
- Tabel objecten.ruimten zit naast de bouwlaag_id ook bouwlaag in de tabel – uit 3.2.0 (rede onbekend).
- Tabel objecten.scenario heeft in 3.2.0. geen geometrie, dit wel overgenomen uit 3.0.4.
- Tabel objecten.veiligh_bouwk verwijst naar objecten.veiligh_bouwk_type voor soort
- Tabel objecten.veiligh_ruimtelijk kolom bijzonderheid – uit 3.2.0.
- Tabel objecten.veilighv_org verwijst naar objecten.veilighv_org_type voor soort
- Tabel objecten.bouwlagen:
  - Tabel objecten.bouwlagen kolom fotografie_id integer – uit 3.0.4.
  - Tabel objecten.bouwlagen kolom bag_afwijking – een bouwlaag kan aangepast worden waardoor deze afwijkt van de BAG contour. Dit wordt bijgehouden in de kolom “geom”. Echter door deze afwijking suggereert de combinatie “geom” + “pand_id” dat dit mogelijk de BAG contour is. Door dit kenmerk “verschil_met_bag” te vullen is dit duidelijk.
  - Het is zaak dat ook objecten.object verrijkt wordt met de geometrie uit objecten.bouwlagen.
  - Extra trigger: Wordt als repressief object in objecten.object gebruikt als geovlak met trigger. NB voeg een dergelijke trigger toe aan het applicatieschema voor objecten.bouwlagen om objecten.object te vullen in het applicatie schema QGIS/COGO Uit 3.0.4.:

Mogelijke oplossingsrichting:

CREATE OR REPLACE RULE object_bouwlaag_ins AS
    ON INSERT TO objecten.bouwlagen
    DO INSTEAD
INSERT INTO objecten.object (geom, omschrijving, object_id)
 (geom, geovlak, basisreg_identifier, formelenaam, bijzonderheden, bron, bron_tabel)
  VALUES (new.geom, new.omschrijving, new.object_id)
 VALUES (st_pointonsurface(new.geom), st_multi(new.geom), new.pand_id, new.formelenaam, new.bijzonderheden, 'BAG', 'Bouwlagen')
   RETURNING id INTO object_id;
    new.id = object_id;
    RETURN new;


- Tabel objecten.terrein:
  - QGIS: Er veranderd niks aan de QGIS kant, vulling van terrein blijft hetzelfde. De werkwijze is nog steeds terrein en dan pand. Wil je echter enkel een pand als repressief object dan moet de tooling aangepast worden.
  - COGO: Er veranderd niks aan de COGO kant. Tabel objecten.terrein is er bij gekomen, die kan gebruikt worden naast objecten.object voor opslag terrein. De keus is er nu wel om hierop in te tekenen.
  - Extra functie inbouwen: De kolom geovlak wordt gevuld met trigger vanuit objecten.terrein naar objecten.object. Een repressief object heeft altijd een punt en geovlak (uit terrein of uit bouwlagen).
  - NB voeg deze trigger toe aan het applicatieschema voor QGIS/COGO. Ik wist niet of de QGIS plug-in gebruik maakt van views, wfs-t of van db tabellen.

Mogelijke oplossingsrichting:

CREATE OR REPLACE RULE object_terrein_ins AS
    ON INSERT TO objecten.object_terrein
    DO INSTEAD
INSERT INTO objecten.object (geom, omschrijving, object_id)
 (geom, geovlak, basisreg_identifier, formelenaam, bijzonderheden, bron, bron_tabel)
  VALUES (new.geom, new.omschrijving, new.object_id)
 VALUES (st_pointonsurface(new.geom), st_multi(new.geom), 'nvt', new.formelenaam, new.bijzonderheden, '', 'Terreinen')
   RETURNING id INTO object_id;
    new.id = object_id;
    RETURN new;


- Tabel objecten.object:
  - Typeobject – uit 3.2.0.
  - Status – verplaatst vanuit objecten.historie --> verwijst naar objecten.object_status_type
  - Teamlid_behandeld_id – verplaatst vanuit objecten.historie
  - Teamlid_afgehandeld_id – verplaatst vanuit objecten.historie
  - Matrix_code – verplaatst vanuit objecten.historie --> verwijst naar  objecten.object_matrix_code
  - Aanpassing – uit 3.2.0.
  - Plus_info (json) – nieuw veld vrij in te vullen
  - Geovlak – uit 3.0.4. wordt gevuld met trigger vanuit terreinen of bouwlagen. 


- Tabel objecten.gevaarlijkestof:
  - De werkwijze is dat een gevaarlijkestof altijd een locatie mee krijgt (elke gevaarlijke stof heeft een opslag).
  - De tabel gevaarlijke stof heeft de volgende samengevoegde velden:
    - Ghs_categorie --> verwijst naar gevaarlijkestof_categorie_type – uit 3.0.4
    - Ghs_divisie – uit 3.0.4
    - Identificatie_gevaar – uit 3.0.4
    - Identificatie_stof – uit 3.0.4
    - gevaarlijkestof_vnnr_id – uit 3.2.0
    - omschrijving – uit 3.2.0
  - Tevens komen de velden van de oude tabel objecten.opslag over naar objecten.gevaarlijkestof:
    - Geom
    - Bouwlaag_id
    - Fotografie_id
    - Rotatie
  - De kolom opslag_id is verwijderd, de opslag tabel bestaat niet meer.
  - Vergt aanpassing in QGIS plug-in: plat slaan van object 1:n maar object met gevaarlijke stof 1:1.

- Tabel objecten.gevaarlijkestof verwijst naar objecten.gevaarlijke_stof_eenheid_type voor eenheid en objecten.gevaarlijke_stof_toestand_type voor toestand.
- Tabel objecten.gevaarlijkestof_schade_cirkel verwijst naar objecten.gevaarlijkestof_schade_cirkel_type voor soort.
- Tabel objecten.beheersmaatregelen bestaat niet in 3.0.4. , wel toegevoegd
- Tabel objecten.beheersmaatregelen_inzetfase bestaat niet in 3.0.4. , wel toegevoegd
- Tabel objecten.gebruiksfunctie bestaat niet in 3.0.4., wel toegevoegd
- Tabel objecten.bereikbaarheid verwijst naar objecten.bereikbaarheid_type voor soort
- Tabel objecten.isolijnen staat er in vanuit Brandweer Friesland
- Tabel objecten.grid vanuit Brandweer Friesland

## Algemene aanpassingen

- TYPES zijn omgezet naar *_type tabellen
- Smallint id’s vervangen met integers
- Omdat niet elke Veiligheidsregio dezelfde applicaties gebruikt wordt er vanaf 4.0.0. een onderscheid gemaakt tussen het datamodel en applicatie specifieke views, functionaliteit naar een aparte schema’s. De schema’s zijn dan bijvoorbeeld:
  **afhankelijk van de regio:**
  - qgis (funtionaliteit, views)
  - moi (views)
  - cogo (funtionaliteit, views)
  - logging (vrije keus voor elke regio, historieopbouw)
  **altijd aanwezig:**
  - objecten (datamodel)

- Schema qgis aangemaakt met daarin de tabellen:
  - qgis.styles – was algemeen.styles
  - qgis.styles_symbols_type – was algemeen.styles_symbols_type
  - qgis.isolijnen – was objecten.isolijnen
  - qgis.grid – was objecten.grid
  - qgis.view_bouwlagen als voorbeeld view
- SERIALS zijn vervangen met IDENTITY en sequences zijn niet meer.


## Overige opmerkingen:

- Audit (logging) functionaliteit zit er niet standaard in. Dit is aan de regio zelf om dit te activeren (bouwlagen b.v. heeft nu geen triggers hier op).
- Afwijkende kolommen. Uitgangspunt in de samenvoeging was een kolom wel mee te nemen als die in één van de modellen zat. Het beoordelen van deze keus zal in een vervolg versie gedaan kunnen worden. Dit om beide versies niet in de weg te zitten door een kolom weg te halen.
- Het was niet duidelijk welke views voor QGIS, MOI of algemene views zijn. Het verzoek is deze onder te verdelen in een apart schema specifiek voor die afnemende applicatie.

Een voorbeeld view voor schema qgis is:

CREATE OR REPLACE VIEW qgis.view_bouwlagen
 AS
 SELECT bl.id,
    bl.geom,
    bl.datum_aangemaakt,
    bl.datum_gewijzigd,
    bl.bouwlaag,
    bl.bouwdeel,
    o.id AS object_id,
    o.formelenaam,
    o.status
   FROM objecten.bouwlagen bl
     JOIN objecten.object o ON bl.id = o.id
  WHERE o.status::text = 'in gebruik'::text AND o.datum_geldig_vanaf <= now() OR o.datum_geldig_vanaf IS NULL AND o.datum_geldig_tot > now() OR o.datum_geldig_tot IS NULL;

ALTER TABLE qgis.view_bouwlagen
    OWNER TO oiv_admin;

GRANT SELECT ON TABLE qgis.view_bouwlagen TO oiv_read;
GRANT INSERT, UPDATE, DELETE ON TABLE qgis.view_bouwlagen TO oiv_write;
GRANT ALL ON TABLE qgis.view_bouwlagen TO oiv_admin;


- Niet alle *_type tabellen zijn gevuld, zoals scenario en matrix i.v.m. afwijkende gegevens per regio / niet bekend welke waardes.
- Een aantal types in 3.2.0. hebben een kolom symbol_name en size. Ik heb die niet meegenomen, dit omdat dit specifiek voor qgis dient. Een mogelijke oplossing is dit in de view aan te passen door de symboolnaam en symbool size daar te noteren. Als er een view gebruikt wordt als input voor QGIS. 
- Bluswater is niet meegenomen, we zijn vanuit OIV IMROI objecten uitgegaan.
