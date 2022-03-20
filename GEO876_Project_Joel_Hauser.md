
# GEO 876 Programming Project Assessment
## Spatial Analysis of Trees in the City of Zurich

### Introduction

This Programming Project addresses the spatial distribution of trees in the city of Zürich. It is studied whether the trees and their distribution varies within the different districts of the city.
The object of this task is not to produce ground breaking insight or be particularly accurate (more on the why later) but to train python coding skills for relative newbies. The data processing and the addressing of the research questions is solved with an object-oriented programming approach.

### Research Questions
The following research questions have been identified and the code is developed towards answering these questions.
The questions have been chosen in a way that the city nursery (Stadtgärtnerei) might benefit of that knowledge.

**Research Questions**
1. Which of the districts (quartiere) contains the most trees?
This could be of relevance for city employees as to where to most resources for maintenance need to be allocated (Trimming of trees and cleaning of leaves)
2. Which of the districts has the highest or lowest tree density?
In fighting the urban heat island trees and parks can lessen the impact and reduce temperatures.
3. Which of the districts has the greatest /lowest variety of tree species?
This can be used as a proxy for biodiversity in a district. Different insect as well bird species prefer different sorts of trees. City gardeners could use this knowledge to take action foster greater biodiversity and renew the tree inventory accordingly.
4. Which district has the oldest trees (mean of tree age)? This Knowledge could be used as to where a lot of old trees have to be replaced soon.
5. Using bounding boxes instead of the polygons of the districts how much are is tree count beeing exaggerated by counting one tree multiple times?

### Data
For this spatial analysis two datasets are being used.
the first one contains the inventory of trees in Zurich. [Baumkataster](https://data.stadt-zuerich.ch/dataset/geo_baumkataster)

The second dataset contains the districts of Zürich. [Vermessungsbezirke](https://data.stadt-zuerich.ch/dataset/geo_quartiere__basierend_auf_vermessungsbezirken_)

Both datasets have been downloaded as json files.
*The data has been accessed and downloaded on March 10. 2022.*

### Methods
To process the datafiles and representing the data, a object-oriented coding approach has been chosen.
Following classes are being used for this task:
- Point
- Tree
- Quartier
- CSVmakr
- Dictmakr

**Point Class**
is the lowest level class and is used to create point objects containing and x and y coordinate. These objects are further used in the classes Tree and Quartier.

The json files containing the coordinates are delivered in WGS84 projection. This projection is based on degrees and thus makes it difficult for calculating areas. Therefore the coordinates are transformed to the swiss CRS CH1903+ /LV95. The transformation follows the approach of H. DUPRAZ (1992). The code used is based on the script provided by the Federal Office of Topography by A. SCHMOCKER (2014). 
>H. Dupraz, Transformation approchée de coordonnées WGS84 en coordonnées nationales suisses, IGEO-TOPO, EPFL, 1992. [LINK](https://www.swisstopo.admin.ch/content/swisstopo-internet/de/online/calculation-services/_jcr_content/contentPar/tabs/items/dokumente_und_publik/tabPar/downloadlist/downloadItems/7_1467103072612.download/ch1903wgs84_d.pdf)

>A. Schmocker, Programmatic computer scripts to transpose GPS WGS-84 coordinates to/from the swiss military and civilian coordinate system (LV-03/CH-1903), Federal Office of Topography swisstopo, Wabern 2014. [LINK](https://github.com/ValentinMinder/Swisstopo-WGS84-LV03)

**Tree Class**
is used to represent the trees and their attributes. Most importantly is its location represented by a object of the point class from above. Furthermore attributes like year of planting and tree species  are used to construct these objects. This class has a report function which returns the attributes from mentioned above. The plant year is converted to age.

**Quartier Class**
is used to represent districts as objects. It takes a list of coordinates (point objects). From the list of points a bounding box is calculated which spatially encapsulates the district in a rectangle.
This class has many methods used for the spatial analysis. These methods can be seen as way to generates attributes of the district regarding the trees it contains. 

**Dictmakr Class**
is used to create a dictionary for a district containing the attributes determined with the methods from the Quartier class. This class is used to aggregate the data for further processing and export.

**CSVmakr Class**
takes a list of dictionaries created with class above and returns and saves the data in a csv file. This file can easily be shared and / or the data processed for the analysis in a different program like Excel or R.
For the purpose of this Project we want to use the processed data right away and therefore this class also has method for creating a pandas dataframe from the dictionaries.


```python
import pandas as pd
import math
import json # to work with json
import datetime

# This class is used to initialise the fundamental spatial objects
# Can be used for Trees and Quartiere
# It has methods to convert the coordinates projected in WGS84 to CH1903+ / LV 95
class Point:
    def __init__(self, lng, lat):
        # Use of conditional statements when initialising the object to make sure Points are not
        # converted more than once
        # Due to False easting and northing easy to detect.
        if lat > 100000:
            self.x = lng
            self.y = lat
        else:
            self.x = self.WGStoCHx(lat, lng)
            self.y = self.WGStoCHy(lat, lng)
        
    # Convert WGS lat/long (° dec) to CH x
    def WGStoCHx(self, lat, lng):
        lat = self.DecToSexAngle(lat)
        lng = self.DecToSexAngle(lng)
        lat = self.SexAngleToSeconds(lat)
        lng = self.SexAngleToSeconds(lng)
        # Axiliary values (% Bern)
        lat_aux = (lat - 169028.66) / 10000
        lng_aux = (lng - 26782.5) / 10000
        x = ((1200147.07 + (308807.95 * lat_aux) + \
            + (3745.25 * pow(lng_aux, 2)) + \
            + (76.63 * pow(lat_aux,2))) + \
            - (194.56 * pow(lng_aux, 2) * lat_aux)) + \
            + (119.79 * pow(lat_aux, 3))
        return int(x)

    # Convert WGS lat/long (° dec) to CH y
    def WGStoCHy(self, lat, lng):
        lat = self.DecToSexAngle(lat)
        lng = self.DecToSexAngle(lng)
        lat = self.SexAngleToSeconds(lat)
        lng = self.SexAngleToSeconds(lng)
        # Axiliary values (% Bern)
        lat_aux = (lat - 169028.66) / 10000
        lng_aux = (lng - 26782.5) / 10000
        y = (2600072.37 + (211455.93 * lng_aux)) + \
            - (10938.51 * lng_aux * lat_aux) + \
            - (0.36 * lng_aux * pow(lat_aux, 2)) + \
            - (44.54 * pow(lng_aux, 3))
        return int(y)
    
    # auxillary methods for the Convertion of WGS to CH
    # Convertion from Degrees to Sexagesimal angle
    def DecToSexAngle(self, dec):
        degree = int(math.floor(dec))
        minute = int(math.floor((dec - degree) * 60))
        second = (((dec - degree) * 60) - minute) * 60
        return degree + (float(minute) / 100) + (second / 10000)
    
    # Convert sexagesimal angle (dd.mmss,ss) to seconds
    def SexAngleToSeconds(self, dms):
        degree = 0 
        minute = 0 
        second = 0
        degree = math.floor(dms)
        minute = math.floor((dms - degree) * 100)
        second = (((dms - degree) * 100) - minute) * 100
        return second + (minute * 60) + (degree * 3600)
    
    # Method for reporting the coordinates of a point
    #uses a thousand separator for better readability
    def report(self):
        return f"({self.x:,} | {self.y:,})"

# Class used to create Tree objects with the attributes of interesst
# Note that one attribute is a Object of the class Point
class Tree:
    def __init__(self, point, baumnamedeu, pflanzjahr, kronendurchmesser):
        self.point = point
        self.baumnamedeu = baumnamedeu
        self.pflanzjahr = pflanzjahr
        self.kronendurchmesser = kronendurchmesser
        
    # for some trees the age is not determined
    # this cases are labelled and excluded from the analysis
    def treeage(self):
        # to make this script work correctly in the future we get the current year via datetime and convert to int
        currentDateTime = datetime.datetime.now()
        date = currentDateTime.date()
        year = date.strftime("%Y")
        if self.pflanzjahr == None:
            return "unknown"
        else:
            age = int(year) - int(self.pflanzjahr)
            return age        
        
    # this mehtods creates a summary of the tree's location and attributes
    def report(self):
        print("This Tree has the coordinates") 
        print(self.point.report())
        print(f"it is a {self.baumnamedeu}")
        print(f"it is {self.treeage()} years old")
        print(f"its crown crosssection is approx. {self.kronendurchmesser} meters")
        print("")

# Class for creating quartier objects with defining bounding and the name of the quartier
# the points argument is a list of the vertex' of the polygong represented as point objects 
class Quartier:
    # when initialising the object, a bounding box (represented by two point objects) is extracted from the list of points given 
    #Here an instance of a Bounding Box defined by two points
    def __init__(self, points, name):
        self.name= name
        self.points = points
        xs = []
        ys = []
        for p in self.points:
            xs.append(p.x)
            ys.append(p.y)
        # here it becomes clear why the initialising needs to be conditional
        # otherwise already converted (WGS84 to CH1903+) will be converted again (result would be complete horsesh*)
        self.ll = Point(min(xs), min(ys))
        self.ur = Point(max(xs), max(ys)) 

    # This method prints the bounding box and spatial insights about the trees in a quartier
    # the argument needed is a list of trees as instances of class tree
    def report(self, trees): # trees nicht vergessen
        print(self.name)
        print('The coordinates of the bounding box are/ the bounding box of the Quartier is:')        
        print(self.ll.report())
        print(self.ur.report())
        print(f"{self.name} contains {self.numberoftreescontained(trees)} trees")
        print(f"is has a tree density of {self.density(trees)} trees per km squared")
        print(f"the mean age of trees (with known age) in {self.name} is {self.meantreeage(trees)} years")
        print(f"this quartier has {self.biodiversity(trees)} unique Tree species")
        print("")
        
    # Calculating area, division to return area in km^2 and rounding to two decimals
    def area(self):
        #return area in km squared
        return round((self.ur.x - self.ll.x) * (self.ur.y - self.ll.y)/ 1000000, 2)
    
    # Mehtod to check if a point (a tree in this case) is contained within the bounding box
    def contains(self, p):
        if p.x < self.ur.x and p.x > self.ll.x and p.y < self.ur.y and p.y > self.ll.y:
             return True
        else:
            return False
    
    # returns the sum of the trees contained within the quartier
    # makes use of the methods above
    # the argument needed is a list of trees as instances of class tree
    def numberoftreescontained(self, trees):
        Ntrees = 0
        for t in trees:
            if self.contains(t.point) == True:
                Ntrees = Ntrees + 1
        return Ntrees
    
    # returns the tree densitivy within the quartier
    # makes use of the method above
    def density(self, trees):
        return round((self.numberoftreescontained(trees)/self.area()),2)
    
    # method to calculate the mean tree age within a quartier
    # conditional statement to exclude Trees that are not within quartier and where the age (pflanzjahr) is unkown
    # the argument needed is a list of trees as instances of class tree
    def meantreeage(self, trees):
        Ntrees = 0
        meanage_aux = 0
        for t in trees:
            if self.contains(t.point) and t.treeage() != "unknown":
                Ntrees = Ntrees + 1
                meanage_aux = meanage_aux + t.treeage()
            else:
                pass
        return round(meanage_aux / Ntrees,2)
    
    # This method counts the unique tree species in a quartier. Proxy for Biodiversity
    # the argument needed is a list of trees as instances of class tree
    def biodiversity(self, trees): 
        unique_trees_aux= []
        for t in trees:
            if self.contains(t.point) == True:
                unique_trees_aux.append(t.baumnamedeu)
        return len(set(unique_trees_aux))

# This class takes a quartier instance and the list of trees ( as instances of class tree)
# This class is used to create a slsightly less dumb report
class Dictmakr:
    def __init__(self, quartier, trees):
        self.quartier = quartier
        self.trees = trees
    def makedict(self, quartier, trees):
        dictquartier = {"name":quartier.name,
                        "bounding box ll": quartier.ll.report(),
                        "bounding box ur": quartier.ur.report(),
                        "area": quartier.area(),
                        "number of trees": quartier.numberoftreescontained(trees),
                        "tree density": quartier.density(trees),
                        "mean tree age": quartier.meantreeage(trees),
                        "unique tree species": quartier.biodiversity(trees)}
        return dictquartier
    
# Class used to create a csv from a list of dictionaries with the aid of a pandas dataframe
# csv is exported and can easily be analyised further in other applications or shared
# also has a method for only returning a pandas dataframe for further analysis within python
class CSVmakr:
    def __init__(self, dict_list):
        self.dict_list = dict_list
        
    # creating a pandas dataframe    
    def makedf(self):
        # this dict doesnt have the entries ordered as wanted
        df_aux = pd.DataFrame.from_dict(self.dict_list)
        # so we rearange
        df= df_aux[["name",
                    "bounding box ll",
                    "bounding box ur",
                    "area",
                    "number of trees",
                    "tree density",
                    "mean tree age",
                    "unique tree species"]]
        return df
    
    # this method creates a csv containing all the data created from the analysis and names it accordingly
    # using the method makedf()
    def makecsv(self):
        df = self.makedf()
        df.to_csv(r"analysis_output_quartiere_ZH", index= False, header= True)
        return print("CSV successfully created")
    

```

**Main Function** is used to read in the two datasets (Trees and Quartiere). It’s used to initiate the objects from the data and thus populates the model. 
It uses the classes and their methods to conduct the spatial analysis and writes the data to lists and finally exports it to a csv file. 
To better show what it does it prints the report of the first tree object as well the report of the first district with its calculated attributes from the list of tree objects.
It also says that the csv file has been created successfully.



```python
#the main function is used to create all the objects from the data, shows the first instances of objects (trees and quartiere)  
# performes the spatial analysis and writes the results in a csv

def main():
    # This opens the json data file containing the baumkataster
    with open("./Baumkataster/data/gsz.baumkataster_baumstandorte.json", encoding='utf-8') as json_file:
        data_BK = json.load(json_file)
    
    # from the json the features containing attributes and geometry are extracted
    trees_data = data_BK['features']
    
    # Processing the extracted features to populate model with objects
    global trees_obj
    trees_obj = []
    for t in trees_data:
        # creating of point instances from the geometry
        x = t['geometry']['coordinates'][0]
        y = t['geometry']['coordinates'][1]
        point_obj = Point(x,y)
        # creating the tree instance with attributes extraxted from json
        baumnamedeu = t["properties"]["baumnamedeu"]
        kronendurchmesser = t["properties"]["kronendurchmesser"]
        pflanzjahr = t["properties"]["pflanzjahr"]
        tree = Tree(point_obj, baumnamedeu, pflanzjahr, kronendurchmesser)
        # appending the list of tree instances
        trees_obj.append(tree)
        
    # to see whats in the List of tree objects       
    for t in trees_obj[:1]:
        t.report()
    
    # This opens the json data file containing the quartiere    
    with open("./Quartiere_ZH/data/stzh.adm_vermbezirke_a.json", encoding='utf-8') as json_file:
        data_Q = json.load(json_file)
    
    quartier_data = data_Q['features']
    
    # Processing the extracted features to populate model with objects
    quartier_obj = []
    for q in quartier_data:
        name_q = q['properties']['name'] 
        # objectifing polygon vertex' to point objects and creating a list with vertex' as point objects
        coordinates_Q = q['geometry']['coordinates'][0] 
        coordinates_obj = []
        for c in coordinates_Q: 
            x = c[0]
            y = c[1]
            vertex = Point(x,y)
            # append the list of vertex'
            coordinates_obj.append(vertex)
            
        # creating the Quartier Object and appending the Quartier list
        # we feed the name and the list of vertex' to it
        # the Quartier object initialises its bounding box from the list of vertex'
        quartier = Quartier(coordinates_obj, name_q)
        quartier_obj.append(quartier)

    # to see whats in the List of quartier objects
    # and see what the result of the spatial analysis is
    for q in quartier_obj[:1]:
        q.report(trees_obj)
    
    # Settup for full spatial analysis on every quartier
    list_quartier_dicts = []
    # iteration through the list of quartier objects from above
    for q in quartier_obj:
        # creating instance of class Dictmakr
        dictmakr_obj = Dictmakr(q, trees_obj)
        
        #appending list containing the results of the spatial analysis in dicts        
        list_quartier_dicts.append(dictmakr_obj.makedict(q, trees_obj))
        
    # Using the CSVmakr class to write the list of dictionaries as a csv file
    CSVmakr(list_quartier_dicts).makecsv()
    
    # also the dataframe will help answering the research question right away
    # the dataframe needs to be a global variable to use it later
    global df_analysis
    df_analysis = CSVmakr(list_quartier_dicts).makedf()
```

### Results


```python
# Call of the main function
main()
```

    This Tree has the coordinates
    (1,251,531 | 2,686,027)
    it is a Berg-Ahorn, Wald-Ahorn
    it is unknown years old
    its crown crosssection is approx. 14 meters
    
    Wiedikon
    The coordinates of the bounding box are/ the bounding box of the Quartier is:
    (1,243,377 | 2,678,931)
    (1,248,342 | 2,682,296)
    Wiedikon contains 15684 trees
    is has a tree density of 938.6 trees per km squared
    the mean age of trees (with known age) in Wiedikon is 34.95 years
    this quartier has 625 unique Tree species
    
    CSV successfully created
    

### Addressing Research Questions

The necessary processing has been completed and the data aggregated in a way that allows the answering of the research questions.

Each RQ is addressed in a cell showing the results of a query on the dataframe.
The results are  discussed in a cell right below the corresponding query output.

**1.Which of the districts (quartiere) contains the most trees?**


```python
# same as above but different row
df_sorted = df_analysis.sort_values(by='number of trees', ascending=0)
print(df_sorted.iloc[[0, -1]])
```

            name          bounding box ll          bounding box ur   area  \
    0   Wiedikon  (1,243,377 | 2,678,931)  (1,248,342 | 2,682,296)  16.71   
    12  Leimbach  (1,241,584 | 2,680,272)  (1,244,279 | 2,681,767)   4.03   
    
        number of trees  tree density  mean tree age  unique tree species  
    0             15684        938.60          34.95                  625  
    12             1742        432.26          27.90                  340  
    

The Wiedikon district contains the most trees (18648) whereas the district with the least amount of trees is the Leimbach district (1742). For maintenance of the trees the city nursery needs to allocate a lot more resources towards Wiedikon than Leimbach. Comparing absolute numbers somewhat limits the insight since the districts vary in size. This is also the case here. Wiedikon is more than 4 times bigger than Leimbach.

To compensate for this it is more useful to look at tree density and get more insights which of the districts is the greenest.

**2. Which of the districts has the highest or lowest tree density?**


```python
# same as above but different row
df_sorted = df_analysis.sort_values(by='tree density', ascending=0)
print(df_sorted.iloc[[0, -1]])

```

               name          bounding box ll          bounding box ur   area  \
    17  Unterstrass  (1,248,132 | 2,682,071)  (1,251,559 | 2,683,740)   5.72   
    6     Hottingen  (1,246,514 | 2,683,860)  (1,249,473 | 2,687,951)  12.11   
    
        number of trees  tree density  mean tree age  unique tree species  
    17             9418       1646.50          31.90                  312  
    6              2848        235.18          39.04                  202  
    

In terms of tree density the relatively small district Unterstrass has the lead (1646 trees per km<sup>2</sup>). The least green district of Zürich is Hottingen (235 trees per km<sup>2</sup>). It is also a relatively large district. The City nursery could together with city planners work out a plan to improve tree density in order to fight the health adverse urban heat island.
*Note that Hottingen has a great portion of its area covered by forest, which are not in the inventory of the Baumkataster. So the disparities are not as grave as they seem in the first place.*

**3. Which of the districts has the greatest /lowest variety of tree species?**


```python
# same as above but different row
df_sorted = df_analysis.sort_values(by='unique tree species', ascending=0)
print(df_sorted.iloc[[0, -1]])

```

            name          bounding box ll          bounding box ur   area  \
    0   Wiedikon  (1,243,377 | 2,678,931)  (1,248,342 | 2,682,296)  16.71   
    13  Fluntern  (1,247,320 | 2,683,829)  (1,249,610 | 2,686,364)   5.81   
    
        number of trees  tree density  mean tree age  unique tree species  
    0             15684        938.60          34.95                  625  
    13             1835        315.83          36.97                  162  
    

Wiedikon is here on top of the list again with 625 unique tree species. This is likely due to the fact that it has relatively large parks cemeteries and family gardens. Birds and Insects will feel pretty accommodated here. The district with the lowest biodiversity is the district Fluntern (162 unique tree species). This number is likely to be incorrect since the district features a lot of single family homes an as well the Zoo. Privately owned tree are not in the tree inventory. So this figure is to be viewed with caution.

**4. Which district has the oldest trees (mean of tree age)?**


```python
# sorting df by mean tree age
df_sorted = df_analysis.sort_values(by='mean tree age', ascending=0)
#the .iloc method shows the first and last row of the sorted dataframe
print(df_sorted.iloc[[0, -1]])
```

                     name          bounding box ll          bounding box ur  area  \
    7            Riesbach  (1,243,932 | 2,683,358)  (1,246,723 | 2,686,710)  9.36   
    20  Industriequartier  (1,248,179 | 2,680,009)  (1,249,894 | 2,683,095)  5.29   
    
        number of trees  tree density  mean tree age  unique tree species  
    7              6418        685.68          41.72                  562  
    20             5876       1110.78          23.21                  211  
    

Riesbach has the oldest trees in the mean and the trees there are almost double the age of trees standing in the Industriequartier (41.72 vs 23.21 years). Note that this figure is likely to be biased because for a lot of trees the age is not determinable. It is assumable that the planting year of older trees is less likely to be unknown than for younger trees. It must be assumed that this bias is spatialy not distributed evenly. Older districts (inner city) have likely a higher percentage of trees with unknown age. So the ranks of the districts must be viewed with caution.

**5. Using bounding boxes instead of the polygons of the districts how much is the tree count beeing exaggerated by counting one tree multiple times?**


```python
# this prints the total number of trees counted within the bounding boxes
# note tree can be contained within more than one bounding box
print("the amount of tree objects contained within one or more quartier bounding box")
print(df_analysis['number of trees'].sum())

# this counts the amount of unique trees in zurich
tot_unique_trees = 0
for t in trees_obj:
    tot_unique_trees += 1

print("the amount of unique trees")
print(tot_unique_trees)    
```

    the amount of tree objects contained within one or more quartier bounding box
    130349
    the amount of unique trees
    75720
    

Due to the usage of rectangular bounding boxes the areas of the districts is exaggerated in every case and most importantly overlapping. This results in counting a tree multiple times for different districts if the bounding boxes are overlapping. Comparing the two numbers in becomes apparent that the bounding box method greatly overcounts trees by almost a factor of 2. Not all districts overcount the trees contained similarly due to the shape of the district borders. The more a district perimeter represents a rectangle the less overcounting happens and also the area deviation is less severe.
