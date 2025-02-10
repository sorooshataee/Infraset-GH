# Infraset-GH
# -*- coding: UTF-8 -*-
# -*- coding: utf-8 -*-
import copy
import os.path
from collections import OrderedDict
from math import sqrt, floor, ceil, log10, isnan

import rhinoscriptsyntax as rs
from Grasshopper import DataTree
from Rhino import Display
from Rhino import Geometry
from System import Drawing
from System import Guid
from System.Drawing import Color

# Material data table (embedded directly in code)
MATERIAL_DATA = [
    {
        "Material": "Steel",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 113578,
        "Embodied_Energy_Coefficient": 1841720,
        "Embodied_Water_Coefficient": 2035420,
        "Life_Span": 100,
        "Density": 7740
    },
    {
        "Material": "Concrete",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 336,
        "Embodied_Energy_Coefficient": 3640,
        "Embodied_Water_Coefficient": 5180,
        "Life_Span": 100,
        "Density": 1400
    },
    {
        "Material": "Cement",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 1950,
        "Embodied_Energy_Coefficient": 16520,
        "Embodied_Water_Coefficient": 10920,
        "Life_Span": 100,
        "Density": 1500
    },
    {
        "Material": "Wood",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 944,
        "Embodied_Energy_Coefficient": 13632,
        "Embodied_Water_Coefficient": 19110,
        "Life_Span": 80,
        "Density": 430
    },
    {
        "Material": "Painting",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 8500,
        "Embodied_Energy_Coefficient": 133200,
        "Embodied_Water_Coefficient": 247000,
        "Life_Span": 1,
        "Density": 1250
    },
    {
        "Material": "Polycarbonate",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 16800,
        "Embodied_Energy_Coefficient": 228000,
        "Embodied_Water_Coefficient": 318000,
        "Life_Span": 25,
        "Density": 1200
    },
    {
        "Material": "uPVC",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 5838,
        "Embodied_Energy_Coefficient": 106097,
        "Embodied_Water_Coefficient": 779390,
        "Life_Span": 35,
        "Density": 1390
    },
    {
        "Material": "Gravel",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 66.24,
        "Embodied_Energy_Coefficient": 897.6,
        "Embodied_Water_Coefficient": 3496,
        "Life_Span": 30,
        "Density": 1840
    },
    {
        "Material": "Sand",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 36,
        "Embodied_Energy_Coefficient": 510,
        "Embodied_Water_Coefficient": 2700,
        "Life_Span": 30,
        "Density": 1500
    },
    {
        "Material": "Asphalt",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 528,
        "Embodied_Energy_Coefficient": 1125.8,
        "Embodied_Water_Coefficient": 7672,
        "Life_Span": 100,
        "Density": 2649
    },
    {
        "Material": "Aluminium",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 79677,
        "Embodied_Energy_Coefficient": 1039096,
        "Embodied_Water_Coefficient": 661408,
        "Life_Span": 50,
        "Density": 2712
    },
    {
        "Material": "Recycled aggregate",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 10.56,
        "Embodied_Energy_Coefficient": 145.2,
        "Embodied_Water_Coefficient": 132,
        "Life_Span": 100,  # Data not provided in table
        "Density": 1320
    },
    {
        "Material": "Hot_rolled_structural_steel",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 22790,
        "Embodied_Energy_Coefficient": 304640,
        "Embodied_Water_Coefficient": 291490,
        "Life_Span": 100,  # Data not provided in table
        "Density": 7850
    },
    {
        "Material": "Concrete_25_MPA",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 240,
        "Embodied_Energy_Coefficient": 3200,
        "Embodied_Water_Coefficient": 6000,
        "Life_Span": 100,  # Data not provided in table
        "Density": 2409
    },
       {
        "Material": "uPVC pipe - 114.3 mm outer dia., 4.85 mm thick",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 5808.4,
        "Embodied_Energy_Coefficient": 106047.9,
        "Embodied_Water_Coefficient": 779041.9,
        "Life_Span": 25,  # Data not provided in table
        "Density": 1400
    },      
       {
        "Material": "uPVC pipe - 225.3 mm outer dia., 11.1 mm thick",
        "Unit": "m³",
        "Embodied_GHG_Coefficient": 5760.0,
        "Embodied_Energy_Coefficient": 105600.0,
        "Embodied_Water_Coefficient": 776800.0,
        "Life_Span": 25,  # Data not provided in table
        "Density": 1400
    }
]


class MaterialDatabase(object):
    def __init__(self):
        self.material_db = {}
        self.populate_material_db()

    def populate_material_db(self):
        for mat in MATERIAL_DATA:
            material_name = mat["Material"]
            self.material_db[material_name] = {
                "Embodied_GHG_Coefficient_(kgCO2e/FU)": mat["Embodied_GHG_Coefficient"],
                "Embodied_Energy_Coefficient_(MJ/FU)": mat["Embodied_Energy_Coefficient"],
                "Embodied_Water_Coefficient_(L/FU)": mat["Embodied_Water_Coefficient"],
                "Life_Span_(Years)": mat["Life_Span"],
                "Density_(kg/m³)": mat["Density"]
            }

    def print_material_db(self):
        print("Material Database (DB):")
        for material, properties in self.material_db.items():
            print("{0}: {1}".format(material, properties))

class ElementDatabase(object):
    def __init__(self, material_db):
        self.material_db = material_db
        self.element_db = {}
        self.painting_factor = 60  # Constant factor for maintenance painting

    def calculate_embodied_metrics(self, material_name, thickness_m):
        # Calculate the volume in m³ (assuming 1 m² area for simplicity)
        volume_m3 = thickness_m * 1
        material_props = self.material_db.get(material_name, {})

        # Calculate embodied metrics using coefficients from the material database
        ghg = volume_m3 * 1000 * material_props.get("Embodied_GHG_Coefficient_(kgCO2e/FU)", 0)
        energy = volume_m3 * 1000 * material_props.get("Embodied_Energy_Coefficient_(MJ/FU)", 0)
        water = volume_m3 * 1000 * material_props.get("Embodied_Water_Coefficient_(L/FU)", 0)

        # Calculate material mass (density * volume)
        density = material_props.get("Density_(kg/m³)", 0)
        material_mass = volume_m3 * density  # mass = volume * density (kg)

        return ghg, energy, water, material_mass

    def populate_elements(self):
        # Define the elements with their respective layers, material thicknesses, and operational data
        elements = {
            "Tram post": {
                "layers": [
                    {"material": "Painting", "thickness_mm": 0.0008},
                    {"material": "Concrete", "thickness_mm": 0.09},
                    {"material": "Steel", "thickness_mm": 0.008},
                    {"material": "Hot_rolled_structural_steel", "thickness_mm": 0.0008}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
            "Light post": {
                "layers": [
                    {"material": "Painting", "thickness_mm": 0.0008},
                    {"material": "Concrete", "thickness_mm": 0.07},
                    {"material": "Hot_rolled_structural_steel", "thickness_mm": 0.053},
                    {"material": "Polycarbonate", "thickness_mm": 0.0025},
                    {"material": "Steel", "thickness_mm": 0.005}
                ],
                "replacement_factor": 0.5,
                "Operational_GHG_(kgCO2e/No)": 2.503233,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
            "Sign post": {
                "layers": [
                    {"material": "Painting", "thickness_mm": 0.2},
                    {"material": "Concrete", "thickness_mm": 36},
                    {"material": "Steel", "thickness_mm": 70}
                ],
                "replacement_factor": 2,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
                        "Railing": {
                "layers": [
                    {"material": "Painting", "thickness_mm": 0.00008},
                    {"material": "Concrete", "thickness_mm": 0.0125},
                    {"material": "Steel", "thickness_mm": 0.0022}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
                        "Planter box": {
                "layers": [

                    {"material": "Concrete", "thickness_mm": 0.0145}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
                        "Underground pipes": {
                "layers": [

                    {"material": "uPVC", "thickness_mm": 0.0066}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
                        "Street pits": {
                "layers": [

                    {"material": "uPVC pipe - 114.3 mm outer dia., 4.85 mm thick", "thickness_mm": 1},
                    {"material": "uPVC pipe - 225.3 mm outer dia., 11.1 mm thick", "thickness_mm": 0.16}
                ],
                "replacement_factor": 1.5,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            }
        }

        # Calculate total embodied metrics and mass for each element
        for element_name, element_data in elements.items():
            total_ghg = total_energy = total_water = total_mass = 0
            material_masses = {
                "Steel": 0, "Concrete": 0, "Asphalt": 0, "Painting": 0, "Gravel": 0,
                "Aluminum": 0, "Wood": 0, "Hot_rolled_structural_steel": 0, "uPVC pipe - 114.3 mm outer dia., 4.85 mm thick": 0,"uPVC pipe - 225.3 mm outer dia., 11.1 mm thick": 0
            }

            # Extract the replacement factor for this element
            replacement_factor = element_data.get("replacement_factor", 1)

            # Iterate through each layer to accumulate metrics
            for layer in element_data["layers"]:
                material = layer["material"]
                thickness_m = layer["thickness_mm"] / 1000  # Convert thickness from mm to m

                # Calculate embodied metrics and mass for the current layer
                ghg, energy, water, mass = self.calculate_embodied_metrics(material, thickness_m)
                total_ghg += ghg
                total_energy += energy
                total_water += water
                total_mass += mass  # Sum the mass for the element

                # Track the mass for specific materials
                if material in material_masses:
                    material_masses[material] += mass

            # Apply the replacement factor to embodied metrics only
            total_ghg 
            total_energy 
            total_water 

            # Include operational metrics from the element data
            operational_ghg = element_data["Operational_GHG_(kgCO2e/No)"]
            operational_energy = element_data["Operational_Energy_(MJ/No)"]
            operational_water = element_data["Operational_Water_(L/No)"]

            # Calculate maintenance metrics including painting
            painting_mass = material_masses["Painting"]
            maintenance_mass = total_mass * replacement_factor + (self.painting_factor * painting_mass)
            maintenance_ghg = total_ghg * replacement_factor+ (self.painting_factor * painting_mass)
            maintenance_energy = total_energy * replacement_factor+ (self.painting_factor * painting_mass)
            maintenance_water = total_water * replacement_factor+ (self.painting_factor * painting_mass)

            # Store the total metrics and operational data in the element database
            self.element_db[element_name] = {
                "Initial_Embodied_GHG_(kgCO2e)": total_ghg,
                "Initial_Embodied_Energy_(MJ)": total_energy,
                "Initial_Embodied_Water_(L)": total_water,
                "Total_Mass_(kg)": total_mass,
                "Maintenance_Mass_(kg)": maintenance_mass,
                "Maintenance_Embodied_GHG_(kgCO2e)": maintenance_ghg,
                "Maintenance_Embodied_Energy_(MJ)": maintenance_energy,
                "Maintenance_Embodied_Water_(L)": maintenance_water,
                "Steel_Mass_(kg)": material_masses["Steel"],
                "Hot_rolled_structural_steel_Mass_(kg)": material_masses["Hot_rolled_structural_steel"],
                "Concrete_Mass_(kg)": material_masses["Concrete"],
                "Asphalt_Mass_(kg)": material_masses["Asphalt"],
                "Painting_Mass_(kg)": material_masses["Painting"],
                "Gravel_Mass_(kg)": material_masses["Gravel"],
                "Aluminum_Mass_(kg)": material_masses["Aluminum"],
                "Wood_Mass_(kg)": material_masses["Wood"],
                "Operational_GHG_(kgCO2e/No)": operational_ghg,
                "Operational_Energy_(MJ/No)": operational_energy,
                "Operational_Water_(L/No)": operational_water
            }

    def create_or_update_element(self, category, data_dict):
        self.element_db[category] = data_dict
        return self.element_db[category]

    def print_element_db(self):
        # Print the contents of the element database
        print("Element Database:")
        for element_name, metrics in self.element_db.items():
            print("{}:".format(element_name))
            for key, value in metrics.items():
                print("  {}: {}".format(key, value))
            print()  # Add a blank line for readability

class AssemblyDatabase(object):
    def __init__(self, material_db):
        self.material_db = material_db
        self.assembly_db = {}
        self.maintenance_factor = 1  # Constant factor for maintenance applied to pavement layer

    def calculate_embodied_metrics(self, material_name, thickness_m):
        # Calculate volume in m³, assuming 1 m² area
        volume_m3 = thickness_m * 1
        material_props = self.material_db.get(material_name, {})

        # Calculate embodied metrics
        ghg = volume_m3 * 1000 * material_props.get("Embodied_GHG_Coefficient_(kgCO2e/FU)", 0)
        energy = volume_m3 * 1000 * material_props.get("Embodied_Energy_Coefficient_(MJ/FU)", 0)
        water = volume_m3 * 1000 * material_props.get("Embodied_Water_Coefficient_(L/FU)", 0)
        
        # Calculate material mass
        density = material_props.get("Density_(kg/m³)", 0)
        material_mass = volume_m3 * density

        return ghg, energy, water, material_mass

    def populate_assembly_db(self):
        # Define assemblies and layers
        assemblies = {
            "Concrete_Highway": {
                "layers": [
                    {"material": "Recycled aggregate", "thickness_mm": 300},
                    {"material": "Recycled aggregate", "thickness_mm": 300},
                    {"material": "Cement", "thickness_mm": 300},
                    {"material": "Cement", "thickness_mm": 100},
                    {"material": "Concrete", "thickness_mm": 100},
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
            "Asphalt_Highway": {
                "layers": [
                    {"material": "Recycled aggregate", "thickness_mm": 300},
                    {"material": "Recycled aggregate", "thickness_mm": 300},
                    {"material": "Gravel", "thickness_mm": 300},
                    {"material": "Asphalt", "thickness_mm": 400},
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0
            },
            "Urban_Street": {
                "layers": [
                    {"material": "Asphalt", "thickness_mm": 0.15},
                    {"material": "Recycled aggregate", "thickness_mm": 0.28},
                    {"material": "Gravel", "thickness_mm": 0.5},
                    {"material": "Sand", "thickness_mm": 0.43}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0,
                "Embodied_GHG_Coefficient_pavement": 0.2,
        "Embodied_Energy_Coefficient_pavement": 1125.8,
        "Embodied_Water_Coefficient_pavement": 7672,
        "replacement_factor_pavement": 1.2
            },
            "Pedestrian_Way": {
                "layers": [
                    {"material": "Concrete_25_MPA", "thickness_mm": 0.04},
                    {"material": "Recycled aggregate", "thickness_mm": 0.0025},
                    {"material": "Gravel", "thickness_mm": 0.3},
                    {"material": "Sand", "thickness_mm": 0.14}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0,
                "Embodied_GHG_Coefficient_pavement": 0.2,
        "Embodied_Energy_Coefficient_pavement": 1125.8,
        "Embodied_Water_Coefficient_pavement": 7672,
        "replacement_factor_pavement": 0.45
            },
            "Bicycle_path": {
                "layers": [
                    {"material": "Concrete_25_MPA", "thickness_mm": 0.04},
                    {"material": "Recycled aggregate", "thickness_mm": 0.0020},
                    {"material": "Gravel", "thickness_mm": 0.3},
                    {"material": "Sand", "thickness_mm": 0.14}
                ],
                "replacement_factor": 1,
                "Operational_GHG_(kgCO2e/No)": 0,
                "Operational_Energy_(MJ/No)": 0,
                "Operational_Water_(L/No)": 0,
                "Embodied_GHG_Coefficient_pavement": 361,
        "Embodied_Energy_Coefficient_pavement": 1125.8,
        "Embodied_Water_Coefficient_pavement": 7672,
        "replacement_factor_pavement": 0.45
            }
        }

        # Calculate metrics for each assembly
        for assembly_name, assembly_data in assemblies.items():
            total_ghg = total_energy = total_water = total_mass = 0
            material_masses = { "Steel": 0, "Concrete": 0, "Asphalt": 0, "Painting": 0, "Gravel": 0, "Aluminum": 0, "Wood": 0,  "Sand": 0}
            replacement_factor = assembly_data.get("replacement_factor", 1)

            # Identify primary pavement layer
            if assembly_name == "Pedestrian_Way" or "Bicycle_path":
            # Explicitly set pavement material to "Concrete" for Pedestrian_Way
                pavement_material = "Concrete_25_MPA"
            else:
            # Default logic: Use "Concrete" or "Asphalt" based on assembly name
                pavement_material = "Concrete" if "Concrete" in assembly_name else "Asphalt"

            pavement_mass = 0



            for layer in assembly_data["layers"]:
                material = layer["material"]
                thickness_m = layer["thickness_mm"] / 1000
                ghg, energy, water, mass = self.calculate_embodied_metrics(material, thickness_m)

                total_ghg += ghg
                total_energy += energy
                total_water += water
                total_mass += mass

                if material in material_masses:
                    material_masses[material] += mass

                # Capture mass of pavement layer for maintenance calculations
                if material == pavement_material:
                    pavement_mass += mass
 
            # Apply the replacement factor to embodied metrics only
            total_ghg *= replacement_factor
            total_energy *= replacement_factor
            total_water *= replacement_factor

            # Retrieve operational metrics
            operational_ghg = assembly_data.get("Operational_GHG_(kgCO2e/No)", 0)
            operational_energy = assembly_data.get("Operational_Energy_(MJ/No)", 0)
            operational_water = assembly_data.get("Operational_Water_(L/No)", 0)

            # Calculate maintenance metrics, using maintenance factor for pavement layer
            Embodied_GHG_Coefficient_pavement = assembly_data.get("Embodied_GHG_Coefficient_pavement", 0)
            Embodied_Energy_Coefficient_pavement = assembly_data.get("Embodied_Energy_Coefficient_pavement", 0)
            Embodied_Water_Coefficient_pavement = assembly_data.get("Embodied_Water_Coefficient_pavement", 0)
            replacement_factor_pavement = assembly_data.get("replacement_factor_pavement", 0) 
            maintenance_mass =  (replacement_factor_pavement * pavement_mass )
            maintenance_ghg = (replacement_factor_pavement * pavement_mass * Embodied_GHG_Coefficient_pavement *1000) 
            maintenance_energy = ( replacement_factor_pavement * pavement_mass * Embodied_Energy_Coefficient_pavement) 
            maintenance_water = ( replacement_factor_pavement * pavement_mass * Embodied_Water_Coefficient_pavement) 

            # Store metrics in the assembly database
            self.assembly_db[assembly_name] = {
                "Initial_Embodied_GHG_(kgCO2e)": total_ghg,
                "Initial_Embodied_Energy_(MJ)": total_energy,
                "Initial_Embodied_Water_(L)": total_water,
                "Total_Mass_(kg)": total_mass,
                "Maintenance_Mass_(kg)": maintenance_mass,
                "Maintenance_Embodied_GHG_(kgCO2e)": maintenance_ghg,
                "Maintenance_Embodied_Energy_(MJ)": maintenance_energy,
                "Maintenance_Embodied_Water_(L)": maintenance_water,
                "Steel_Mass_(kg)": material_masses["Steel"],
                "Concrete_Mass_(kg)": material_masses["Concrete"],
                "Asphalt_Mass_(kg)": material_masses["Asphalt"],
                "Painting_Mass_(kg)": material_masses["Painting"],
                "Gravel_Mass_(kg)": material_masses["Gravel"],
                "Aluminum_Mass_(kg)": material_masses["Aluminum"],
                "Wood_Mass_(kg)": material_masses["Wood"],
                "Sand_Mass_(kg)": material_masses["Sand"],
                "Operational_GHG_(kgCO2e/No)": operational_ghg,
                "Operational_Energy_(MJ/No)": operational_energy,
                "Operational_Water_(L/No)": operational_water
            }

    def print_assembly_db(self):
        # Print all assembly data
        print("Assembly Database:")
        for assembly_name, metrics in self.assembly_db.items():
            print("{}: {}".format(assembly_name, metrics))

            
class InfrastructureDatabase(object):
    def __init__(self, assembly_db):
        self.assembly_db = assembly_db
        self.infrastructure_db = {}

    def populate_infrastructure_db(self):
        highways = ["Concrete Highway", "Asphalt Highway"]
        for highway in highways:
            assembly_data = self.assembly_db.assembly_db.get(highway, {})
            replacement_factor = 2  # Example factor for infrastructure

            total_ghg = assembly_data.get("Initial_Embodied_GHG_(kgCO2e)", 0) * replacement_factor
            total_energy = assembly_data.get("Initial_Embodied_Energy_(MJ)", 0) * replacement_factor
            total_water = assembly_data.get("Initial_Embodied_Water_(L)", 0) * replacement_factor

            self.infrastructure_db[highway] = {
                "Initial_Embodied_GHG_(kgCO2e)": total_ghg,
                "Initial_Embodied_Energy_(MJ)": total_energy,
                "Initial_Embodied_Water_(L)": total_water
            }

    def print_infrastructure_db(self):
        print("Infrastructure Database:")
        for key, value in self.infrastructure_db.items():
            print("{0}: {1}".format(key, value))

            
class InfrastructureDatabase(object):
    def __init__(self, assembly_db):
        self.assembly_db = assembly_db
        self.infrastructure_db = {}

    def populate_infrastructure_db(self):
        city_road_energy = self.assembly_db.assembly_db.get("Concrete Highway", {}).get("Initial_Embodied_Energy_(MJ)", 0)
        city_road_water = self.assembly_db.assembly_db.get("Concrete Highway", {}).get("Initial_Embodied_Water_(L)", 0)

        self.infrastructure_db["Highway"] = {
            "Initial_Embodied_Energy_(MJ)": city_road_energy * 2,
            "Initial_Embodied_Water_(L)": city_road_water * 50
        }

    def print_infrastructure_db(self):
        print("Infrastructure Database:")
        for key, value in self.infrastructure_db.items():
            print("{0}: {1}".format(key, value))
# Instantiate and populate Material Database
material_db = MaterialDatabase()
material_db.print_material_db()

# Instantiate and populate Element Database
element_db = ElementDatabase(material_db.material_db)
element_db.populate_elements()
element_db.print_element_db()

# Instantiate and populate Assembly Database
assembly_db = AssemblyDatabase(material_db.material_db)
assembly_db.populate_assembly_db()
assembly_db.print_assembly_db()


# Instantiate and populate Infrastructure Database
infrastructure_db = InfrastructureDatabase(assembly_db)
infrastructure_db.populate_infrastructure_db()
infrastructure_db.print_infrastructure_db()
