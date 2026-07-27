The current directory, /home/js/LightRAG-Dev/jrs, is a subdirectory of LightRAG-Dev.
The LightRAG-Dev project uses retrieval augmented generation via the LightRAG library to create ai troubleshooting experts. 
Typically, I task LightRAG with indexing technical manuals and then I query the index to get troubleshooting advice.
I have tried ingesting electrical schematics in PDF format but LightRAG does not do very well at accurately indexing these.
The following PDF file is a good example of an electrical schematic that I am trying to index: 
/home/js/LightRAG-Dev/jrs/work/mod_linx/mod_linx_data/PS20115MLM4-2.pdf

I have 3 methods in mind for indexing schematics that might get better results:
1. Use OpenCV to create lines and figures on the drawings in such a way as to enhance the features in the drawings more understandable to an ai.
2. Train an ai model using YOLO on OpenCV enhanced drawings such that the trained model can read symbols on drawing previously not seen by the model.
3. Modify the skills found in the following folder for the purpose of reading electrical schematics.
   /home/js/LightRAG-Dev/jrs/construction_skills
   The hope is to use the skills to get a textual description of all the entities on the schematic and all the physical connections, logical connections and relationships between all the entities. Then use the text documents created as the source documents to be indexed by LightRAG. Then it is hoped that the index can be queried about the schemeatic and that accurate, helpful answers will be returned.
   The newly created skills paterned after what is found in /home/js/LightRAG-Dev/jrs/construction_skills should reside in the following directory: 
   /home/js/LightRAG-Dev/jrs/schematic_skills 

Method 3 is the only approach we are working on right now. Methods 1 and 2 might be considered at a later time but are not the focus of current work.

Please examine /home/js/LightRAG-Dev/jrs/_claude_notes/schematic_indexing_plan.md

Please read all the files which are given as recommended reading in /home/js/LightRAG-Dev/jrs/_claude_notes/schematic_indexing_plan.md 

The following is my request:  
Notice item 1 of section 9. "Open Items / Next Steps" of /home/js/LightRAG-Dev/jrs/_claude_notes/schematic_indexing_plan.md  
It says the following:  
"1. **Draft the extraction skill** in `schematic_skills/` targeting the §5 schema
   (netlist tracer + tiled vision verification). *Not started — awaiting go-ahead. This is
   the intended next action.*"

Please look at the skill found in /home/js/LightRAG-Dev/jrs/construction_skills/SKILL.md and adapt it to extract information from electrical schematics.  
Place this new skill in /home/js/LightRAG-Dev/jrs/schematic_skills.

Next, we will test this new skill by using it to preprocess the electical schematic.  
Then the json file produced by the skill will be indexed by LightRAG and we will then query the LightRAG index to see how well LightRAG can reason about the schematic. 

We will evaluate how well information is extracted from the electrical schematic using the new skill and we will evaluate how well LightRAG is able to reason about the electrical schematic before proceeding to the next steps.
