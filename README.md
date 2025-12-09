# Mise-en-page-sur-QGIS-gr-ce-Python
Ce code permet de générer une mise en page d'Atlas des espaces verts sur QGIS avec Python des communes limitrophes de la Mer Méditerranée.
# ETAPE 1 : IMPORT DES COUCHES ET FILTRAGE
import os
from qgis.core import *

# Chemin du dossier contenant les fichiers GeoPackage
monCheminDeBase = r"D:\Mermedi\Data"

# Charger les couches GeoPackage
communes = QgsVectorLayer(os.path.join(monCheminDeBase, "Communes.gpkg|layername=Communes"), "Communes", "ogr")
mer_medi = QgsVectorLayer(os.path.join(monCheminDeBase, "MerMedi.gpkg|layername=MerMedi"), "Mer Méditerranée", "ogr")

# Vérifier la validité
print("Communes valide :", communes.isValid())
print("Mer Méditerranée valide :", mer_medi.isValid())

if not all([communes.isValid(), mer_medi.isValid()]):
    raise Exception("Erreur de chargement d'une ou des couches — vérifie les fichiers ou les noms internes.")

# Appliquer un filtre sur le champ "name"
mer_medi.setSubsetString('"name" = \'Mediterranean Region\'')
print(" Filtre appliqué sur Mer Méditerranée : name = 'Mediterranean Region'")

# Ajouter les couches au projet
project = QgsProject.instance()
project.addMapLayer(communes)
project.addMapLayer(mer_medi)

# Définir le SCR du projet (ex : WGS 84)
crs = QgsCoordinateReferenceSystem.fromEpsgId(4326)
project.setCrs(crs)
project.setEllipsoid('EPSG:4326')

# Rafraîchir la carte
iface.mapCanvas().refresh()

print(" Couches 'Communes' et 'Mer Méditerranée (filtrée)' ajoutées au projet avec succès !")

                                                  ***

#Etape 2 : Intersection 

from qgis.core import *
import processing

# Récupérer les couches existantes dans le projet
project = QgsProject.instance()
communes = project.mapLayersByName("Communes")[0]
mer_medi = project.mapLayersByName("Mer Méditerranée")[0]

# Utiliser l'algorithme select by location
processing.run("native:selectbylocation", {
    'INPUT': communes,
    'PREDICATE': [0],  # 0 = intersects
    'INTERSECT': mer_medi,
    'METHOD': 0        # 0 = create new selection
})

# Créer une nouvelle couche mémoire avec les communes sélectionnées
selected_fids = [f.id() for f in communes.getSelectedFeatures()]
communes_touche_mer = communes.materialize(QgsFeatureRequest().setFilterFids(selected_fids))
communes_touche_mer.setName("Communes_Touchant_MerMediterranee")

# Ajouter la nouvelle couche au projet
project.addMapLayer(communes_touche_mer)

print("✅ Nouvelle couche 'Communes_Touchant_MerMediterranee' créée avec succès !")

                                ***
                                
#Etape 3 : Symbologie 

from qgis.core import *

# Récupérer les couches existantes dans le projet
project = QgsProject.instance()
mer_medi = project.mapLayersByName("Mer Méditerranée")[0]
communes_touche_mer = project.mapLayersByName("Communes_Touchant_MerMediterranee")[0]

# --- Mer Méditerranée en bleu ---
symbol_mer = QgsFillSymbol.createSimple({
    'color': '102,178,255,150',        # bleu semi-transparent (RVB + alpha)
    'outline_color': 'black',
    'outline_width': '0.8'         # contour moyen
})
mer_medi.renderer().setSymbol(symbol_mer)
mer_medi.triggerRepaint()
iface.layerTreeView().refreshLayerSymbology(mer_medi.id())

# --- Communes touchant la Méditerranée : transparent avec contour noir ---
symbol_communes = QgsFillSymbol.createSimple({
    'color': '255,255,255,0',      # transparent
    'outline_color': 'black',
    'outline_width': '0.8'         # contour moyen
})
communes_touche_mer.renderer().setSymbol(symbol_communes)
communes_touche_mer.triggerRepaint()
iface.layerTreeView().refreshLayerSymbology(communes_touche_mer.id())

print(" Symbologie appliquée : Mer Méditerranée (bleu) et Communes (contour noir).")


                                        ***

#Etape 4 : Espaces verts grace à API overpass turbo exemple Nice 
import requests, json
from qgis.core import QgsVectorLayer, QgsProject, QgsFeature, QgsGeometry, QgsField, QgsPointXY
from PyQt5.QtCore import QVariant

query = """
[out:json][timeout:180];

area["name"="Nice"]["boundary"="administrative"]->.nice;

(
  way["leisure"~"park|garden|nature_reserve|recreation_ground|playground"](area.nice);
  way["landuse"~"forest|grass|village_green"](area.nice);
  way["natural"~"wood|grassland"](area.nice);

  relation["leisure"~"park|garden|nature_reserve|recreation_ground|playground"](area.nice);
  relation["landuse"="forest"](area.nice);
  relation["natural"~"wood|grassland"](area.nice);
);
out geom;
"""

url = "https://overpass-api.de/api/interpreter"
r = requests.post(url, data={'data': query})

if r.status_code != 200:
    raise Exception("Requête Overpass échouée : " + str(r.text))

data = r.json()

layer = QgsVectorLayer("Polygon?crs=EPSG:4326", "Espaces_Verts_Nice", "memory")
pr = layer.dataProvider()
pr.addAttributes([
    QgsField("osm_id", QVariant.String),
    QgsField("type", QVariant.String),
    QgsField("name", QVariant.String)
])
layer.updateFields()

def build_geom(elem):
    if "geometry" not in elem:
        return None

    coords = [(p["lon"], p["lat"]) for p in elem["geometry"]]

    if elem["type"] == "way":
        return QgsGeometry.fromPolygonXY([[QgsPointXY(x, y) for x, y in coords]])

    return None  # (Relations option à ajouter si nécessaire)

for elem in data["elements"]:
    if elem["type"] not in ("way", "relation"):
        continue
    geom = build_geom(elem)
    if not geom:
        continue
    feat = QgsFeature()
    feat.setGeometry(geom)
    feat.setAttributes([
        str(elem["id"]),
        elem["type"],
        elem.get("tags", {}).get("name", "")
    ])
    pr.addFeature(feat)

layer.updateExtents()
QgsProject.instance().addMapLayer(layer)

print(" Espaces verts de Nice ajoutés dans QGIS")

                                            ***

#Etape 5 : Mettre une symbologie sur les espaces verts fraichement créés

from qgis.core import QgsFillSymbol, QgsSimpleFillSymbolLayer, QgsProject

# Récupérer la couche résultat
layer_result = QgsProject.instance().mapLayersByName("Espaces_Verts_Nice_Intersect")[0]

# Créer un symbole de remplissage vert
symbol = QgsFillSymbol.createSimple({
    'color': '0,255,0,100',  # RGBA : vert avec transparence
    'outline_color': '0,0,0',  # contour noir
    'outline_width': '0.3'
})

# Appliquer le symbole à la couche
layer_result.renderer().setSymbol(symbol)
layer_result.triggerRepaint()

print(" Symbologie verte appliquée aux espaces verts")



                                            ***
#Etape 6 : Mise en page propre avec Atlas 

from qgis.core import *
from qgis.PyQt.QtGui import QFont

project = QgsProject.instance()
manager = project.layoutManager()

# --- 1) Récupérer les couches visibles du Canvas ---
root = project.layerTreeRoot()
visible_layers = [l.layer() for l in root.findLayers() if l.isVisible()]

if not visible_layers:
    raise SystemExit("❌ Aucune couche visible dans le canvas !")

# --- 2) Création layout ---
layout_name = "Carte_Espaces_Verts_Nice"

for l in manager.printLayouts():
    if l.name() == layout_name:
        manager.removeLayout(l)

layout = QgsPrintLayout(project)
layout.initializeDefaults()
layout.setName(layout_name)
manager.addLayout(layout)

# --- 3) Carte (avec toutes les couches visibles) ---
item_map = QgsLayoutItemMap(layout)
item_map.setRect(10, 10, 200, 150)

item_map.setLayers(visible_layers)
item_map.setExtent(iface.mapCanvas().extent())

layout.addLayoutItem(item_map)
item_map.attemptMove(QgsLayoutPoint(10, 30, QgsUnitTypes.LayoutMillimeters))
item_map.attemptResize(QgsLayoutSize(190, 150, QgsUnitTypes.LayoutMillimeters))

# --- 4) Titre ---
title = QgsLayoutItemLabel(layout)
title.setText("Espaces verts de la Ville de Nice")
title.setFont(QFont("Arial", 22))
title.adjustSizeToText()
layout.addLayoutItem(title)
title.attemptMove(QgsLayoutPoint(10, 5, QgsUnitTypes.LayoutMillimeters))

# --- 5) Sous-titre ---
subtitle = QgsLayoutItemLabel(layout)
subtitle.setText("Données issues d’OpenStreetMap – Analyse SIG")
subtitle.setFont(QFont("Arial", 12))
subtitle.adjustSizeToText()
layout.addLayoutItem(subtitle)
subtitle.attemptMove(QgsLayoutPoint(10, 18, QgsUnitTypes.LayoutMillimeters))

# --- 6) Légende ---
legend = QgsLayoutItemLegend(layout)
legend.setTitle("Légende")
legend.setLinkedMap(item_map)
legend.setAutoUpdateModel(True)
layout.addLayoutItem(legend)
legend.attemptResize(QgsLayoutSize(50, 50, QgsUnitTypes.LayoutMillimeters))
legend.attemptMove(QgsLayoutPoint(205, 150, QgsUnitTypes.LayoutMillimeters))

# --- 7) Flèche du nord ---
north = QgsLayoutItemPicture(layout)
north.setPicturePath(QgsApplication.defaultThemePath() + "/north_arrows/layout/default.svg")
layout.addLayoutItem(north)
north.attemptResize(QgsLayoutSize(12, 12, QgsUnitTypes.LayoutMillimeters))
north.attemptMove(QgsLayoutPoint(180, 5, QgsUnitTypes.LayoutMillimeters))

# --- 8) Échelle graphique ---
scalebar = QgsLayoutItemScaleBar(layout)
scalebar.setStyle('Single Box')
scalebar.setLinkedMap(item_map)
scalebar.setUnits(QgsUnitTypes.DistanceMeters)
scalebar.setNumberOfSegments(4)
scalebar.setUnitsPerSegment(100)
scalebar.setUnitLabel("m")
layout.addLayoutItem(scalebar)
scalebar.attemptMove(QgsLayoutPoint(10, 185, QgsUnitTypes.LayoutMillimeters))

# --- 9) Auteur ---
auteur = QgsLayoutItemLabel(layout)
auteur.setText("Auteur : Adama Dior SOW, 2025")
auteur.setFont(QFont("Arial", 10))
auteur.adjustSizeToText()
layout.addLayoutItem(auteur)
auteur.attemptMove(QgsLayoutPoint(10, 200, QgsUnitTypes.LayoutMillimeters))

# --- 10) Source ---
zone_texte = QgsLayoutItemLabel(layout)
zone_texte.setText(
    "Source : IGN")
zone_texte.setFont(QFont("Arial", 9))
zone_texte.setFrameEnabled(True)   
zone_texte.adjustSizeToText()
layout.addLayoutItem(zone_texte)
zone_texte.attemptMove(QgsLayoutPoint(10, 270, QgsUnitTypes.LayoutMillimeters))


# --- 11) Ouvrir directement dans le Designer ---
iface.openLayoutDesigner(layout)

print("Mise en page créée")

