# Analiza danych satelitarnych o nocnym oświetleniu Ziemi w R"

## Materiały

[Paczka z kompletnymi materiałami jest też tu](http://datascience.wne.uw.edu.pl/DSS2019.zip) - na GitHibie nie zmieścił się jeden duży plik rastrowy.

[A tu jest output z pliku `Pwojcik_dss2019.Rmd` w formacie html](http://datascience.wne.uw.edu.pl/dss2019.html)

## Autor i prowadzący: Piotr Wójcik

* adiunkt na Wydziale Nauk Ekonomicznych UW
* pomysłodawca i animator [Data Science Lab](http://dslab.wne.uw.edu.pl) na WNE UW
* ekspert w zakresie wykorzystania oprogramowania R oraz SAS do przetwarzania danych oraz zaawansowanego modelowania
* od 2008 r. kierownik i wykładowca studiów podyplomowych „Metody statystyczne w biznesie. Warsztaty z oprogramowaniem SAS”: [http://mswb.wne.uw.edu.pl](http://mswb.wne.uw.edu.pl)
* od 2017 r. kierownik i wykładowca studiów podyplomowych „Data Science w zastosowaniach biznesowych. Warsztaty z wykorzystaniem programu R”: [http://datascience.wne.uw.edu.pl](http://datascience.wne.uw.edu.pl)
* wieloletnie doświadczenie zawodowe analityka ilościowego w branży finansowej, telekomunikacyjnej i badań marketingowych,
* współautor książki „Metody ilościowe w R. Aplikacje ekonomiczne i finansowe”

## Intensywność świateł nocnych -- dane

* dane o NTLI opierają się na **zdjęciach satelitarnych** gromadzonych i przetwarzanych przez National Oceanic and Athmosferic Administration
* **NOAA** udostępnia dwa rodzaje danych:
    * [Version 4 DMSP-OLS](https://ngdc.noaa.gov/eog/dmsp/downloadV4composites.html) -- uśrednione dane roczne dla okresu 1992--2013
    * [Version 1 VIIRS](https://www.ngdc.noaa.gov/eog/viirs/download_dnb_composites.html) -- dane miesięczne począwszy od kwietnia 2012 i uśrednione roczne (tylko 2015 i 2016)
* natężenie świateł mierzone jest dla pikseli o wymiarach 30×30 (DMSP-OLS) lub 15x15 (VIIRS) sekund kątowych
* odpowiada to **mniej niż $1~km^2$** w okolicach równika (około $0.5~km^2$ w przypadku Polski)
* dla każdego piksela NTLI podawane jest w jednostkach nazywanych *digital numbers* (**DN**) na skali 0--63 (DMSP-OLS) lub 0--16384 (VIIRS)
* dane mogą być **agregowane** do poziomu dowolnych jednostek terytorialnych

## Ograniczenia danych o NTLI

* **ograniczenie skali pomiaru** dla DMSP-OLS -- niemożliwe rozróżnienie między natężeniem światła w centrach miast i na peryferiach
* **natężenia światła o małej intensywności** mogą w procesie filtrowania być **wyzerowane** -- nie ma pewności, że wartość 0 oznacza brak oświetlenia -- **wartości DN 1 i 2 są niedoreprezentowane** w danych
* pomiary z różnych lat / satelitów / typów nie są bezpośrednio porównywalne


## Pakiet sf -- *simple features*

Pakiet `sp` wraz z dodatkowymi narzędziami z pakietów `rgeos` i `rgdal` od wielu lat są standardowymi narzędziami służącymi do analizy danych przestrzennych w R. Niedawno pojawiła się alternatywa w postaci pakietu `sf`, który pozwala przechowywać dane przestrzenne w formie zbliżonej do zbiorów danych nie zawierającymi informacji przestrzennych. Ułatwia to analizy danych przestrzennych i umożliwia zastosowanie nowoczesnych efektywnych narzędzi przetwarzania i wizualizacji danych zgodnych z podejściem `tidyverse` (`dplyr`, `ggplot2`) również w przypadku danych przestrzennych. Pakiet `sf` łączy w jednym miejscu funkcjonalności dostępne wcześniej w trzech różnych pakietach `sp`, `rgdal` i `rgeos`. Dlatego szybko zyskuje popularność wśród osób zajmujących się analizami przestrzennymi w R, stając się nowym standardem do tego typu analiz.

Nazwa pakietu `sf` jest skrótem od *simple features* (nie ma polskiego odpowiednika tego terminu) -- nazwy standardu dla danych wektorowych stworzonego wspólnie przez Otwarte Konsorcjum Geoprzestrzenne (ang. *Open Geospatial Consortium*, OGC) oraz Międzynarodową Organizację Normalizacyjną (ang. *International Organization for Standardization*, ISO). Standard ten (ISO 19125-1:2004) określa sposób reprezentowania w komputerowych bazach danych rzeczywistych obiektów (ang. *features*) -- głównie dwuwymiarowych danych wektorowych, ze szczególnym uwzględnieniem ich geometrii. Obiekty mogą mieć przypisane atrybuty opisowe (typu numerycznego, tekstowego lub logicznego -- np. wysokośc punktu nad poziomem morza, wielkość populacji regionu, nazwa miasta, informacja czy droga reprezentowana przez linie jest autostradą, itp). Oprócz tego każdy obiekt ma atrybut związany z jego geometrią. Geometria obiektów definiowana jest w dwuwymiarowym układzie współrzędnych i wykorzystuje linearną interpolację odległości między współrzędnymi. Jako możliwe typy geometrii obiektów przestrzennych występują: punkt, linia, wielobok, wiele punktów, wiele linii, itp. 

Standard *simple features* jest powszechnie stosowany w otwartych przestrzennych bazach danych (takich jak np. PostGIS), czy komercyjnych narzędziach GIS (np. ESRI ArcGIS). Z kolei standard `GeoJSON` jest uproszczoną wersją standardu *simple features*.

## Klasa `sf` -- specjalna ramka danych

W pakiecie `sf` zdefiniowana jest nowa klasa obiektów przestrzennych, mająca taką samą nazwę jak pakiet, czyli `sf`. Klasa `sf` pozwala przechowywać obiekty przestrzenne wszystkich typów. Obiekty klasy `sf` są de facto ramkami danych (`data.frame`, `tibble`) zawierającymi dodatkowo dane przestrzenne. Pozwala to, jak już wspomniano wcześniej, łatwo i efektywnie przetwarzać tego typu dane z wykorzystaniem funkcjonalności pakietu `dplyr`. Możliwa jest także wizualizacja tak zapisanych danych przestrzennych z wykorzystaniem funkcji pakietu `ggplot2`, choć odpowiednie geometrie `geom_sf` dostępne są póki co (kwiecień 2019) jedynie w wersji deweloperskiej pakietu `ggplot2`.

Wiersze tej specjalnej ramki danych zawierają kolejne obiekty przestrzenne (*features*), natomiast kolumny ich atrybuty -- jak w standardowej ramce danych. Obiekty klasy `sf` zawierają jednak specjalną kolumnę o nazwie `geometry`, definiującą geometrie poszczególnych obiektów. Kolumna ta jest obiektem klasy `sfc`, który z kolei jest listą obiektów klasy `sfg`, reprezentujących geometrie (kształty) poszczególnych obiektów (wierszy danych). Klasa `sfg` zawiera te same podstawowe informacje, co poszczególne pola (sloty) obiektów klasy `Spatial` z pakietu `sp`: CRS, współrzędne geograficzne i typ geometrii (kształt obiektu). Spośród siedemnastu zdefiniowanych rodzajów geometrii do najczęściej wykorzystywanych należą^[Szczegółowy opis wszystkich typów geometrii znajduje się np. na stronie https://r-spatial.github.io/sf/articles/sf1.html.]:

* *POINT* -- punkt,
* *MULTIPOINT* -- wiele punktów,
* *LINESTRING* -- linia: sekwencja dwóch lub więcej punktów połączonych liniami prostymi,
* *MULTILINESTRING*  -- wiele linii,
* *POLYGON* -- wielobok: zamknięta figura składająca się z odcinków, która może mieć puste obszary w środku,
* *MULTIPOLYGON* -- wiele wieloboków,
* *GEOMETRYCOLLECTION* -- dowolna kombinacja powyższych typów.

Teoretycznie obiekty klasy `sf` mogą zawierać więcej niż jedną kolumnę określającą geometrię obiektów, jednak zazwyczaj taka kolumna jest tylko jedna.

Dla ułatwienia identyfikacji funkcji z tego pakietu ich nazwy zaczynają się od przedrostka `st_`, co ułatwia autouzupełnianie ich nazwy w RStudio.

Więcej szczegółow dotyczących pakietu `sf` można znaleźć w Pebesma (2018), "Simple Features for R: Standardized Support for Spatial Vector Data"", The R Journal Vol. 10/1, s. 439--446 albo Lovelace i in. (2019), "Geocomputation with R".


## Źródła i dodatkowe materiały

* [nighttime lights calibration](https://damien-c-jacques.rbind.io/post/nighttime-lights-calibration)
* [Spatialdata.pdf](https://rspatial.org/spatial/Spatialdata.pdf) - rozdział 8
* Darmowe mapy shp: [https://www.gadm.org](www.gadm.org)
* [pliki shapefile dla krajów UE z Eurostatu](https://ec.europa.eu/eurostat/web/gisco/geodata/reference-data/administrative-units-statistical-units/nuts#nuts16)
