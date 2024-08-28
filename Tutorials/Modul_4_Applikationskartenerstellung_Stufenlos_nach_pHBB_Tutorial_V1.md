``` r
# License: CC-BY-NC-SA


# Load packages ----
```

``` r
library(pH)
library(raster)
library(leaflet)
library(sf)
library(ggplot2)
library(ggpubr)
source("RFunctions/calcTotalAmount.R")
source("RFunctions/aggAppRatesAB2.R")
source("RFunctions/gg_multiplot_vr.R")
source("RFunctions/VDLUFA_pHBB_r.R")
source("RFunctions/VDLUFA_pHBB_adj.R")
```

# Schlaginformationen

Diese Information ist für alle Module notwendig und sollte immer zuerst
eingegeben werden. Der Import der Schlaggrenze (Struktur) ist auch
Pflicht, um den Lagebezug festzulegen.

``` r
# Schlaginformationen ----
farm <- "PP"
field <- 1392
schlag.sf <- st_read("Daten/1392/Feldgrenze/PP_1392_Schlaggrenze_Neu.shp")
```

    ## Reading layer `PP_1392_Schlaggrenze_Neu' from data source 
    ##   `C:\HNE-Ingmar\Vorlesungen\2023\Versuchswesen Pflanzenbau\pH-BB_Toolbox\Daten\1392\Feldgrenze\PP_1392_Schlaggrenze_Neu.shp' 
    ##   using driver `ESRI Shapefile'
    ## Simple feature collection with 1 feature and 5 fields
    ## Geometry type: POLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: 463068.4 ymin: 5804461 xmax: 464284.1 ymax: 5805426
    ## Projected CRS: ETRS89 / UTM zone 33N

``` r
# Wo soll das Ergebnis gespeichert werden 
path <- "Daten/1392/output/Streukarten/"  # define output path
```

## Schlaggrenze mit Leaflet anzeigen

``` r
schlag.wgs84 <- st_transform(schlag.sf, crs=4326)
leaflet() %>%
  addProviderTiles(providers$Esri.WorldImagery) %>%
  addPolygons(data = schlag.wgs84,
              fillOpacity=0.5, stroke=F)
```

![](Modul_4_Applikationskartenerstellung_Stufenlos_nach_pHBB_Tutorial_V1_files/figure-markdown_github/unnamed-chunk-4-1.png)

# Parameter Applikationskartenerstellung

## Berechnungsmethode für Kalkung

Angabe der verwendeten Methode zur Berechnung des Kalkbedarfs. Hier wird
der **stufenlose Algorithmus nach pH-BB** verwendet. Das Ergebnis wird
später als Datei mit der Endung “stufenlos-pHBB” gespeichert.

``` r
model <- "stufenlos-pHBB"  
```

## Kalkungsintervall in Jahren

**Numeric Input** (als Integer; Default = 4; ansonsten stehen Werte
zwischen 1 - 10 zur Auswahl)

``` r
## Kalkungsintervall ----
kiv <- 4  
```

## Jahr der letzten pH-Wert-Bodenuntersuchung

**Numeric Input** (Jahr als Integer (z.B. 2023))

``` r
## Jahr pH-Wert-Bodenuntersuchung ----
ph_date <- 2021  # siehe Jahr der eingeladenen pH-Rasterkarte
```

## Arbeitsbreite des Streuers in Meter

**Numeric Input** (als Integer; Default = 12; ansonsten stehen Werte
zwischen 1 bis 40 zur Auswahl). Die Pixel der gerechnetetn
Applikationskarte besitzen die Kantenlänge mit der angegebenen
Arbeitsbreite.

``` r
## Arbeitsbreite ----
w_width <- 12 
```

## Ausgabe der Applikationskarte

Die Applikationskarte kann für einen beliebigen Kalkdünger (KD)
berechnet werden. Dies erfolgt über den Parameter `lime_conversion`.
Default ist `FALSE` und bedeutet das die Menge in CaO \[kg/ha\]
ausgegeben wird. Bei `TRUE`, müssen der Name des Kalkdüngers (**Text
Input**) und die Mengen der basischen Bestandteile von CaCO<sub>3</sub>
und MgCO<sub>3</sub> in Prozent angegeben werden (Werte 1 bis 100). Der
Neutralisationswert und Umrechnungsfaktor des KD wird automatisch
berechnet.

``` r
## Umrechnung CaO in KD ---- 
```

``` r
lime_conversion <- FALSE   # oder TRUE 

if (lime_conversion == TRUE){
  
  # Eingabe Parameter KD
  fertilizer <- "Dolokorn"      # Names des Kalkduengers (KD) 
  caco3 <- 60                   # Gehalt des KD an CaCo3 in %
  mgco3 <- 30                   # Gehalt des KD an MgCo3 in %
  
} else {
  fertilizer <- "CaO"
}
```

## Ausrichtung der Applikationskarte an AB-Linie (Fahrspur)

Liegen feste Fahrspuren im Betrieb vor, kann die Applikationskarte an
die **Fahrspur (AB Linie)** ausgerichtet werden. Die Applikationskarte
wird automatisch rotiert und neu zentriert, so dass die Fahrspur genau
in der Mitte eines Pixels verläuft.  
Als Auswahl stehen hier:

-   **ohne Ausrichtung** mit `align <- F`
-   **mit Ausrichtung an Fahrspur** mit `align <- T` Für die Ausrichtung
    wird ein Shapefile benötigt, welches die festen Fahrspuren des
    Schlags enthält.

``` r
## Fahrspurausrichtung ----
align <- F # FALSE: ohne Ausrichtung; TRUE: mit Ausrichtung an Fahrspur    

if (align == T) {
  ab_line <- sf::st_read("Daten/1392/ABLinien/PP_1392_AB_Linie.shp") # Shapefile Import
}
```

## Anpassung der Ausbringmengen (Streukarte)

In Abhängigkeit des verwendeten Kalkstreuers sind eventuell Anpassungen
an der Streukarte notwendig. Verschiedene Einstellungen können hier
vorgenommen werden, um die Pixelwerte der Streukarte anzupassen.

-   `min_cao` definiert die minimale Ausbringmenge \[kg/ha\] eines
    Pixels
-   `max_cao` definiert die maximale Ausbringmenge \[kg/ha\] eines
    Pixels
-   `zero_val` Nullwerte werden auf gewünschten Wert \[kg/ha\] angehoben

``` r
# Schieberegler
min_cao_down <- TRUE  

# Minimalmenge in [kg/ha] (Optional)  
min_cao <- 0   # oder 500, wenn alle Werte <= 500 auf Null gesetzt werden sollen. 

if (min_cao_down == TRUE){
  min_down <- min_cao
  min_up <- NULL
} else {
  min_up <- min_cao
  min_down <- NULL
}

# Anhebung von Nullwerten in [kg/ha] (Optional) 
zero_val <- NULL            # NULL: keine Anhebung oder INTEGER Wert setzen     

# Maximalmenge in [kg/ha] (Optional)
max_cao <- NULL  # NULL: keine Anhebung oder INTEGER Wert setzen           
```

# Kartengrundlagen als GeoTiff importieren

Kartengrundlagen müssen als Rasterdaten vorliegen. In Abhängigkeit der
ausgewählten **Art der Berechnung** werden unterschiedliche
Eingangsdaten gefordert.

**stufenlos nach pH-BB**:

-   Karte VDLUFA Bodengruppe (GeoTiff)  
-   Karte Humusgehalt (GeoTiff)
-   Karte pH-Wert (GeoTiff)
-   Karte Tongehalt (GeoTiff)

``` r
# Geodaten import ----
 
## Karte VDLUFA-Bodengruppe ----
vdlufa_r <- raster("Daten/1392/Bodenkarten/PP_1392_VDLUFA_BG_EPSG_25833_20211021_2m.tif")

## Karte Humusgehalt ----
humus_r <- raster("Daten/1392/Bodenkarten/PP_1392_Humus.tif")

## Karte pH-Wert ----
ph_r <- raster("Daten/1392/Bodenkarten/PP_1392_pH_2021-08-04.tif")
#ph_r <- ph_r-2  # Beispiel um pH-Werte zu verringern 

## Karte Tongehalt ----
ton_r <- raster("Daten/1392/Bodenkarten/PP_1392_Ton_MLR_PRED_EPSG_25833_20211021_2m.tif")
```

# Applikationskarten-Workflow

## CaO-Bedarfskarte \| Stufenlos-nach-pHBB Algorithmus

Berechnung der CaO-Mengen mithilfe des *Stufenlos nach pH-BB
Algorithmus* mit der Funktion `VDLUFA_phBB_r`. Das Ergebnis ist eine
CaO-Karte in \[kg/ha\] in der gewünschten Auflösung (Default ist 2x2
Meter).

`VDLUFA_phBB(soil, som , ph, clay, mask, res = c(2, 2), width = 0)`

-   `soil:` Raster object containing the VDLUFA soil group \[integer
    values between 1-6\].
-   `som:` som Raster object containing the soil organic matter content
    in % \[0-100\].
-   `ph:` Raster object containing the pH value with CaCl2 method
    (calibrated pH map).
-   `clay:` Raster object containing the clay content in % \[0-100\].
-   `mask:` object of class sf or sp that defines the boundary of a
    field. The projection (CRS) should be in UTM coordinates.
-   `res:` vector of two positive integers. To, optionally, set the
    output resolution of the cells in each direction (horizontally and
    vertically) in meter (unit). The default setting is c(2, 2).\`
-   `width:` numeric; buffer distance to adapt the coverage of the
    RasterStack according to the extend of the `mask` object. The
    default value is 0.\`

``` r
# Stufenlos Algorithmus ----
cao_sfl <- VDLUFA_phBB_r(soil = vdlufa_r, som = humus_r, ph = ph_r, clay = ton_r, mask = schlag.sf) 

#cao_sfl$CaO   # CaO-Raster   
```

## Höchstmengen prüfen

**Achtung**: Wenn die zu streuende Höchstmenge überschritten wird,
erfolgt eine Gabenteilung und zwei Bedarfskarten werden berechnet.

``` r
cao_sfl_adj <- VDLUFA_pHBB_adj(cao = cao_sfl, liv = kiv, year = ph_date, mask = schlag.sf)
```

    ## Es erfolgt ein Gabenteilung, da die empfohlene Hoechstmenge je Kalkung Ueberschritten wird.
    ## Die CaO Menge wird auf zwei Karten aufgeteilt.
    ## Gesamtmenge CaO fuer Karte1 (CaOApp1): 118.4 [t] und fuer Karte2 (CaOApp2): 0.1  [t] und der ZF: 0.75

``` r
# >> Beispiel Höchstmenge wird überschritten <<

# Zweite Gabe auch auf Höchstmenge prüfen
if (nlayers(cao_sfl_adj) > 1){
  cao_app2 <- cao_sfl
  cao_app2$CaO <- cao_sfl_adj$CaOAPP2 / 100
  cao_app3_adj <- VDLUFA_pHBB_adj(cao = cao_app2, 
                                  liv = 4, 
                                  year = as.integer(format(Sys.Date(),"%Y")), 
                                  mask = schlag.sf)
  if (nlayers(cao_app3_adj) > 1){
    cao_sfl_adj$CaOAPP3 <- cao_app3_adj$CaOAPP2
  }
}
```

    ## Die auszubringende Gesamtmenge CaO (CaOApp1) betraegt: 0.1 [t] und der ZF: 0

## Kalk-Streukarte berechnen

``` r
if (lime_conversion == TRUE){
  for (i in 1:nlayers(cao_sfl_adj)){
    l_names <- names(cao_sfl_adj)
    cao_sfl_adj[[i]] <- LimeFertilizerQuantity(x = cao_sfl_adj[[i]], CaCO3 = caco3, MgCO3 = mgco3)
    names(cao_sfl_adj) <- l_names
  } 
}  

# Streukarte berechnen mit Anpassung der Ausbringmengen ----
```

``` r
if (align == F){
  
  lst_sf_ab <- NULL
  
  # Streukarte ohne Ausrichtung ----
  lst <- list() 
  lst_sf_north <- list()
  
  for (i in 1:nlayers(cao_sfl_adj)) {
    cao_sk_r <- aggAppRates(x = cao_sfl_adj[[i]], 
                            boundary = schlag.sf, 
                            cellsize = c(w_width, w_width))
    cao_sk_r[[1]] <- round(cao_sk_r[[1]]/100,0) *100
    
    if (!is.null(min_down)){
      cao_sk_r[cao_sk_r < min_down] <- 0
    }
    
    if (!is.null(min_up)){
      cao_sk_r[cao_sk_r < min_up] <- min_up
    }
    
    if (!is.null(zero_val)){
      cao_sk_r[cao_sk_r <= 0] <- zero_val
    }
    
    if (!is.null(max_cao)){
      cao_sk_r[cao_sk_r > max_cao] <- max_cao
    }
    
    lst[[i]] <- cao_sk_r
    
    cao_sp <- rasterToPolygons(x = cao_sk_r)
    cao_sf <- st_as_sf(cao_sp)
    st_crs(cao_sf) <- st_crs(schlag.sf)
    names(cao_sf)[1] <- "CaOApp"
    lst_sf_north[[i]] <- cao_sf
  }
  
  lst_sk <- lst
  cao_sk_st <- stack(lst)
  names(cao_sk_st) <- paste0(rep("CaOApp",nlayers(cao_sk_st)), c(1:nlayers(cao_sk_st)))
  
} else {
  
  cao_sk_st <- NULL     
  
  # Streukarte mit Ausrichtung ----
  lst <- list() 
  
  for (i in 1:nlayers(cao_sfl_adj)) {
    
    cao_sk_ab <- aggAppRatesAB2(x = cao_sfl_adj[[i]], 
                                abLine = ab_line, 
                                boundary = schlag.sf, 
                                cellsize = c(w_width, w_width) )
    cao_sk_ab[[2]] <- round(cao_sk_ab[[2]]/100,0) * 100
    
    if (!is.null(min_down)){
      cao_sk_ab[cao_sk_ab[[2]] < min_down ,2] <- 0
    }
    
    if (!is.null(min_up)){
      cao_sk_ab[cao_sk_ab[[2]] < min_up, 2] <- min_up
    }
    
    if (!is.null(zero_val)){
      cao_sk_ab[cao_sk_ab[[2]] <= 0, 2] <- zero_val
    }
    
    if (!is.null(max_cao)){
      cao_sk_ab[cao_sk_ab[[2]] > max_cao, 2] <- max_cao
    }
    
    lst[[i]] <- cao_sk_ab
    #summary(cao_sk_ab)
  }
  
  lst_sf_ab <- lst
  for (i in 1:nlayers(cao_sfl_adj)) {
    names(lst_sf_ab[[i]])[2] <- paste0("CaOApp")
  }
}
```

# Benötigte Gesamtmenge Kalk für den Schlag berechnen

Die benötigte Gesamtmenge Kalk in Tonnen soll im Fenster angezeigt
werden.

``` r
# Gesamtmenge in Tonnen ----
if (!is.null(cao_sk_st)){
  amount <- c() 
  for (i in 1:nlayers(cao_sk_st)){
    amount[i] <- round(calcTotalAmount(cao_sk_st[[i]]),0)
  }
}
```

    ## Fuer den gesamten Schlag muessen 122.6 Tonnen Kalk bestellt werden.

    ## Fuer den gesamten Schlag muessen 0 Tonnen Kalk bestellt werden.

``` r
if (!is.null(lst_sf_ab)){
  amount <- c() 
  for (i in 1:length(lst_sf_ab)){
    sf_ab <- lst_sf_ab[[i]]
    sf_ab$area <- round(st_area(sf_ab),1)
    amount[i] <- round(sum((sf_ab$area / 10000) * sf_ab$CaO) / 1000, 0)
  }
}

paste("Für den gesamten Schlag müssen", amount, "Tonnen", fertilizer ,"bestellt werden.")
```

    ## [1] "Für den gesamten Schlag müssen 123 Tonnen CaO bestellt werden."
    ## [2] "Für den gesamten Schlag müssen 0 Tonnen CaO bestellt werden."

``` r
# Path for output files ----
path <- "Daten/1392/output/Streukarten/"

if (!is.null(cao_sk_st)){
  
  cao_sk_sp <- rasterToPolygons(x = cao_sk_st)
  cao_sk_sf <- st_as_sf(cao_sk_sp)
  st_crs(cao_sk_sf) <- st_crs(schlag.sf)
  
  if (lime_conversion == TRUE){
    lime <- fertilizer
  } else {
    lime <- "CaO"
  }
  
  if (nlayers(cao_sk_st) > 1) {
    
    gabe <- paste0(rep("Gabe",nlayers(cao_sk_st)), c(1:nlayers(cao_sk_st)))
    file_name <- paste0(farm, "_", field, "_", lime,"_Streukarte_",w_width, "x", w_width, "_", kiv, "_", model, "_",
                        gabe, ".shp")
    
    for (i in 1:nlayers(cao_sk_st)){
      st_write(cao_sk_sf[i], paste0(path, file_name[i]))
    }
    
  } else {
    
    file_name <- paste0(farm, "_", field, "_CaO_Streukarte_",w_width, "x", w_width, "_", kiv, "_", model, ".shp")
    st_write(cao_sk_sf[i], paste0(path, file_name))
  }  
} 
```

    ## Warning: st_crs<- : replacing crs does not reproject data; use st_transform for that

    ## Writing layer `PP_1392_CaO_Streukarte_12x12_4_stufenlos-pHBB_Gabe1' to data source 
    ##   `Daten/1392/output/Streukarten/PP_1392_CaO_Streukarte_12x12_4_stufenlos-pHBB_Gabe1.shp' using driver `ESRI Shapefile'
    ## Writing 4671 features with 1 fields and geometry type Polygon.
    ## Writing layer `PP_1392_CaO_Streukarte_12x12_4_stufenlos-pHBB_Gabe2' to data source 
    ##   `Daten/1392/output/Streukarten/PP_1392_CaO_Streukarte_12x12_4_stufenlos-pHBB_Gabe2.shp' using driver `ESRI Shapefile'
    ## Writing 4671 features with 1 fields and geometry type Polygon.

``` r
if (!is.null(lst_sf_ab)) {
  
  if (lime_conversion == TRUE){
    lime <- fertilizer
  } else {
    lime <- "CaO"
  }
  
  if (length(lst_sf_ab) > 1) {
    
    gabe <- paste0(rep("Gabe",length(lst_sf_ab)), c(1:length(lst_sf_ab)))
    file_name <- paste0(farm, "_", field, "_", lime,"_Streukarte_",w_width, "x", w_width, "_", kiv, "_", model, "_",
                        gabe, "_AB.shp")
    
    for (i in 1:length(lst_sf_ab)){
      st_write(lst_sf_ab[[i]], paste0(path, file_name[i]))
    }
    
  } else {
    
    file_name <- paste0(farm, "_", field, "_CaO_Streukarte_",w_width, "x", w_width, "_", kiv, "_", model
                        ,"_CaOApp1_AB.shp")
    st_write(lst_sf_ab[[1]], paste0(path, file_name))
  }
} 
```

# Darstellung der Karten

``` r
if (align == F){
  
  gg_multiplot_vr(x = cao_sk_st, name = lime, cols = 1)
  
} else {
  
  plot.list <- list()
  
  for (i in 1:length(lst_sf_ab)) {
    sk_sf <- lst_sf_ab[[i]]
    nm <- names(sk_sf[2]) 
    plot.list[[i]] <- 
      ggplot() +
      geom_sf(data = sk_sf, aes_string(fill =  nm[1]), alpha = 0.8)+
      scale_fill_gradient(low = "yellow", high = "blue", na.value="transparent")+
      theme_void()+
      coord_sf(datum = st_crs(25833))+
      labs(title = paste(nm[1], "|", lime, "[kg/ha]"), fill = "")+
      theme(plot.title = element_text(size=11,  hjust = 0.5))
  }
  
  multiplot(plotlist = plot.list, cols = 1)
}
```

![](Modul_4_Applikationskartenerstellung_Stufenlos_nach_pHBB_Tutorial_V1_files/figure-markdown_github/unnamed-chunk-20-1.png)

``` r
# Anzeige mit mapview ----
if (align == F){
  mapview::mapview(lst_sf_north, zcol = "CaOApp", 
                   #col.regions = colorspace::sequential_hcl(7, "Lajolla"),
                   map.types = c("Esri.WorldImagery", "OpenStreetMap"),
                   burst = TRUE, 
                   layer.name = paste0("CaOApp", 1:length(lst_sf_north)))
} else {
  mapview::mapview(lst_sf_ab, zcol = "CaOApp", 
                   map.types = c("Esri.WorldImagery", "OpenStreetMap"),
                   burst = TRUE, 
                   layer.name = paste0("CaOApp", 1:length(lst_sf_ab))) 
}
```

![](Modul_4_Applikationskartenerstellung_Stufenlos_nach_pHBB_Tutorial_V1_files/figure-markdown_github/unnamed-chunk-21-1.png)

``` r
# Ende ----

#rmarkdown::render("Modul_4_Applikationskartenerstellung_Stufenlos_nach_pHBB_Tutorial_V1.R", encoding = "utf-8", c("html_document"))
#rmarkdown::render("Modul_4_Applikationskartenerstellung_VDLUFA_Rahmenschema.R", c("pdf_document"))
# knitr::spin("Modul_4_Applikationskartenerstellung_VDLUFA_Rahmenschema.R", knit = F)
```