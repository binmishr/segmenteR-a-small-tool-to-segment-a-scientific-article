# segmenteR-a-small-tool-to-segment-a-scientific-article

segmenteR
===============

  Hi everyone ! Whether it is to perform systematic review, some data extraction, or a really specific project that requires it, you may need to extract a particular section from an scientific article in pdf format. If that is the case, you may be curious in this (prototype) R package, segmenteR, a tool to extract a section, for example, “material and methods”, from the pdf of an article, using the fonts information from the pdf and natural language processing.
  Context

  It has been elaborated in the context of a research work conducted at the Joint Research Centre, Directorate F – Health, Consumers and Reference Materials, Ispra (VA), as a sub-part of a project that aimed to analyse a corpus of 801 articles, obtained from the PubMed MeSH database and related to several toxicity topics (cardiotoxicity, genotoxicity, etc).

  We needed to extract both the material and methods section and the results section of each articles, to evaluate the quality of the reporting inside each articles and parse the texts for specific toxicity effects. The work has been published in the Journal of Applied Toxicology, Toxicity effects of nanomaterials for health applications: how automation can support systematic review of the literature ? Blanka Halamoda-Kenzaoui, Etienne Rolland, Jacopo Piovesan, Antonio Puertas Gallardo, Susanne Bremer-Hoffmann doi.org/10.1002/jat.4204.

  While this tool is a prototype, it has a small benchmark to evaluate its performances, as shown inside the article.
  Requirement

  To extract the informations on the fonts inside the pdf we use Poppler, the PDF rendering library and its cpp API. SegmenteR require a version of poppler >= 0.89 as well as a recent version of pdftools. The dev version of pdftools integrate the required change, but you need to install it from github :

  devtools::install_github("ropensci/pdftools") 
  devtools::install_github("ec-jrc/jrc_f2_refine", subdir="segmenteR") 

  Getting started
  The short way

  Download an open access article that was part of the corpus :

  url <- ('https://www.cell.com/action/showPdf?pii=S1525-0016%2816%2931594-5')
  download.file(url, 'Abrams, M T et al 2010.pdf')

  We need a model and from the library udpipe to tokenize and annotate the text :

  ## got the model for annotation
  dl <- udpipe::udpipe_download_model("english-gum")
  str(dl)
  'data.frame':   1 obs. of  5 variables:
   $ language        : chr "english-gum"
   $ file_model      : chr "/tmp/RtmpSJ4sIF/https://casualr.netlify.app//posts/segmenteR_post/english-gum-ud-2.5-191206.udpipe"
   $ url             : chr "https://raw.githubusercontent.com/jwijffels/udpipe.models.ud.2.5/master/inst/udpipe-ud-2.5-191206/english-gum-u"| __truncated__
   $ download_failed : logi FALSE
   $ download_message: chr "OK"
  model <- udpipe::udpipe_load_model(file = dl$file_model)
  #model
  library(segmenteR)
  ## basic example code

  section_aliases <- c("material", "method", "experimental", "experiment", "methodology")

  #model definition can be skipped, the function can download it automatically
  material_and_methods <- segmenteR::extract_section_from_pdf(pdf_name="Abrams, M T et al 2010.pdf",
                                                               udpipe_model=model, 
                                                               section_aliases=section_aliases)

  head(unique(material_and_methods$sentence))
  [1] "MAterIAls And Methods Animals."                                                                                                                                                                           
  [2] "Female Crl:CD-1/ICR mice were obtained from Charles River (Wilmington, MA) and were between 6 and 10 weeks old at time of study (25–30 g)."                                                               
  [3] "All studies were performed in Merck Research Laboratories′"                                                                                                                                               
  [4] "AAALAC-accredited West Point, PA animal facility using protocols approved by the Institutional Animal Care and Use Committee."                                                                            
  [5] "Liposome assemblies, siRNAs, and reagents."                                                                                                                                                               
  [6] "siRNA Lipid Nanoparticles were assembled using a process involving simultaneous mixing of the lipid mixture in an ethanol solution with an aqueous solution of siRNA, followed by stepwise diafiltration."

  And shazam, you have (hopefully) your material and methods section in ConLL-U format inside the dataframe material_and_methods, a format suitable for parsing, etc. You can stop reading this blog entry here.
  A more in-depth example

  This example show the inner working of the function extract_section_from_pdf(), and some functions you made need :

  pdf_name <- "Abrams, M T et al 2010.pdf"
  remove_bibliography <- TRUE

  txt_pdf <- tabulizer::extract_text(pdf_name) # read the text from the pdf
  txt_pdf <- segmenteR::preprocess_article_txt(txt_pdf)

  The role of the function annotate_txt_pdf() is to load the required model and use the library udpipe to tokenize and annotate the text. Please refer to the vignette or the excellent website of udpipe to get more details on the Conll-U format. The reason for this annotation is that we will need it to estimate where the section titles are the most likely placed. For example, if it is the first word of a sentence, if the the word of the sentence is also a section title, etc, it is probably a section title.

  conllu_df <- segmenteR::annotate_txt_pdf(txt_pdf, udpipe_model=model ) # create the dataframe for NLP using udpipe
  head(conllu_df)
    doc_id paragraph_id sentence_id
  1   doc1            1           1
  2   doc1            1           1
  3   doc1            1           1
  4   doc1            1           1
  5   doc1            1           1
  6   doc1            1           1
                                                                                sentence
  1 original article© The American Society of Gene & Cell Therapy Molecular Therapy vol.
  2 original article© The American Society of Gene & Cell Therapy Molecular Therapy vol.
  3 original article© The American Society of Gene & Cell Therapy Molecular Therapy vol.
  4 original article© The American Society of Gene & Cell Therapy Molecular Therapy vol.
  5 original article© The American Society of Gene & Cell Therapy Molecular Therapy vol.
  6 original article© The American Society of Gene & Cell Therapy Molecular Therapy vol.
    token_id    token    lemma  upos xpos                     feats
  1        1 original original   ADJ   JJ                Degree=Pos
  2        2 article© article©  NOUN   NN               Number=Sing
  3        3      The      the   DET   DT Definite=Def|PronType=Art
  4        4 American American PROPN  NNP               Number=Sing
  5        5  Society  Society PROPN  NNP               Number=Sing
  6        6       of       of   ADP   IN                      <NA>
    head_token_id  dep_rel deps misc
  1             2     amod <NA> <NA>
  2            13 compound <NA> <NA>
  3             5      det <NA> <NA>
  4             5     amod <NA> <NA>
  5            13 compound <NA> <NA>
  6             7     case <NA> <NA>

  The other informations we use, and the reason why we work directly on a pdf instead of a text, is the fonts information from the pdf, the font and the fontsize of the words inside the pdf. To do this we use poppler, a PDF rendering library and the cpp API of poppler. We extract this informations using a specific version of pdftools, reason why the package need a version of poppler > 0.89 as well as a recent version of pdftools.

  poppler_output <- segmenteR::prepare_poppler_output(pdf_name)
  head(poppler_output)
        Word             Font Size
  1        © AOJZVN+StoneSans  6.5
  2      The AOJZVN+StoneSans  6.5
  3 American AOJZVN+StoneSans  6.5
  4  Society AOJZVN+StoneSans  6.5
  5       of AOJZVN+StoneSans  6.5
  6     Gene AOJZVN+StoneSans  6.5

  This informations is used to identify the probable font of the section, by first looking at the font used for the words Reference and Acknowledgment, that usually appear in only one occurrence in scientific articles :

  font_section <- segmenteR::identify_font(poppler_output)
  print(font_section)
  [1] "VMUQDX+ITCStoneSans-Semibold"

  Knowing this, we can know which sections are inside the articles and in which order they appear. The list under is the sections titles that the function will try to identify in the poppler output :

  list_of_sections <- list(
      c("Introduction", "INTRODUCTION"),
      c("Materials", "Material", "materials", "material", "MATERIALS", "MATERIAL"),
      c("Methods", "Method", "methods", "method", "METHODS", "METHOD"),
      c("Acknowledgements", "Acknowledgments", "ACKNOWLEDGEMENTS", "ACKNOWLEDGMENTS",
        "Acknowledgement", "Acknowledgment", "ACKNOWLEDGEMENT", "ACKNOWLEDGMENT"),
      c("References", "REFERENCES"),
      c("Results", "RESULTS"),
      c("Discussion", "DISCUSSION", "discussion"),
      c("Abstract", "ABSTRACT"),
      c("Conclusions", "Conclusion", "CONCLUSION", "CONCLUSIONS"),
      c("Background", "BACKGROUND"),
      c("Experimental", "EXPERIMENTAL", "Experiment"),
      c("Supplementary", "SUPPLEMENTARY"),
      c("Methodology"),
      c("Appendix"),
      c("Section", "SECTION")
    )

  Clean_font_txt() remove the most common font inside the articles, which improve the correct localization of the sections by create_section_title_df() inside the pdf.

  poppler_output <- segmenteR::clean_font_txt(poppler_output)
  head(poppler_output)
        Word             Font Size
  1        © AOJZVN+StoneSans  6.5
  2      The AOJZVN+StoneSans  6.5
  3 American AOJZVN+StoneSans  6.5
  4  Society AOJZVN+StoneSans  6.5
  5       of AOJZVN+StoneSans  6.5
  6     Gene AOJZVN+StoneSans  6.5

  Section_title_df is a dataframe that contain the section titles in the article and their relative order, based on the fonts information retrieved from the pdf. This informations (order and existence) will be used to localize the section in the ConLL-U format. This step is needed as the order and the composition of the sections title can change from one article to the other.

  section_title_df <- segmenteR::create_section_title_df(font_section, list_of_sections, poppler_output)
  section_title_df <- segmenteR::clean_title_journal(pdf_name, section_title_df)
  section_title_df <- segmenteR::ad_hoc_reorder(section_title_df)
  head(section_title_df)
                  Word                         Font Size
  293     Introduction VMUQDX+ITCStoneSans-Semibold 10.0
  1321         Results VMUQDX+ITCStoneSans-Semibold 10.0
  5243      Discussion VMUQDX+ITCStoneSans-Semibold 10.0
  6214       Materials VMUQDX+ITCStoneSans-Semibold 10.0
  6216         Methods VMUQDX+ITCStoneSans-Semibold 10.0
  7188 Acknowledgments VMUQDX+ITCStoneSans-Semibold  9.2

  Removing the bibliography prevent some error in the localization in some sections, especially if a reference start with the word “material”. This option can be set to false.

  if (remove_bibliography == TRUE) {
    conllu_df <- segmenteR::remove_bibliography_from_conllu(conllu_df, section_title_df)
    section_title_df <- segmenteR::remove_reference_section_from_titles(section_title_df)
  }

  Knowing the relative order of the sections from one side, their names and the informations from the Conll-U dataframe (position inside the sentence, or the other words in the sentence) we can estimate the position of the different sections inside the Conll-U dataframe. Please note that the positions_sections_df is not the section_title_df, since section_title_df refer to the position inside the output from poppler, while section_title_df indicate the position inside the Conll-U dataframe.

  positions_sections_df <- segmenteR::locate_sections_position_in_conllu(conllu_df, section_title_df)
  segmenteR::check_sections_df(positions_sections_df)
  head(positions_sections_df)
                  section occurrences
  1          Introduction         239
  2               Results        1173
  3            Discussion        6295
  4 Materials and Methods        7563
  6       Acknowledgments        8769
  section <- segmenteR::extract_section_from_conllu(conllu_df, positions_sections_df, section_aliases)
  head(unique(section$sentence))
  [1] "MAterIAls And Methods Animals."                                                                                                                                                                           
  [2] "Female Crl:CD-1/ICR mice were obtained from Charles River (Wilmington, MA) and were between 6 and 10 weeks old at time of study (25–30 g)."                                                               
  [3] "All studies were performed in Merck Research Laboratories′"                                                                                                                                               
  [4] "AAALAC-accredited West Point, PA animal facility using protocols approved by the Institutional Animal Care and Use Committee."                                                                            
  [5] "Liposome assemblies, siRNAs, and reagents."                                                                                                                                                               
  [6] "siRNA Lipid Nanoparticles were assembled using a process involving simultaneous mixing of the lipid mixture in an ethanol solution with an aqueous solution of siRNA, followed by stepwise diafiltration."

  Finally extract_section_from_conllu() provide the section in ConLL-U format inside the dataframe section.
