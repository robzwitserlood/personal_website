---
title: Computer vision on space-born images
date: 2024-06-01
---

# Introduction

## Situation 

The growth of impervious surfaces in urban areas increases their vulnerability to extreme rainfall and high temperatures (Kennisportaal Klimaatadaptatie). However, there is currently no explicit data available that indicates the imperviousness of surfaces in urban areas. Nevertheless, there exists a dataset called the "BGT," which can be used to infer imperviousness (Kadaster). Moreover, satellite images and aerial photos with centimeter-level resolution and national coverage are freely accessible (PDOK, NSO). Previous research has demonstrated the effectiveness of computer vision algorithms in classifying pixels in air-borne and space-born images as either impervious or non-impervious (Readar, Natuur en Milieu).

## Complication 

At PBL, the suitability of image recognition as a technique for developing indicators remains uncertain. It is crucial to understand the reliability of the technique and the types of errors that may occur, along with their magnitude. Additionally, it is essential to determine the consistency of errors in both space and time. Furthermore, the reproducibility of results and the sensitivity of the outcomes to changes in imagery fed into the algorithm need to be assessed. Lastly, it is important to evaluate the expressiveness of the results and their ability to accurately reflect the risks associated with drought and flooding.

## Question 

The research question of this pilot is:

To what extent can impervious surfaces in urban areas be identified based on satellite images/aerial photos and impervious surfaces extracted from the BGT data set with Computer Vision techniques?

The main question is broken down into three sub-questions to be able to structure the answer to the main question, see Table 1. The questions are labelled as “Extraction question”, “Reproduction question” and “Generalization question” to facilitate referencing in this report.

|     |     |     |
| --- | --- | --- |
| #   | Label | Sub-question |
| 1   | Extraction question | To what extent can impervious and non-impervious surfaces be extracted from the BGT data set? |
| 2   | Reproduction question | To what extent can impervious and non-impervious surfaces, extracted from the BGT data set, be reproduced with Computer Vision techniques? |
| 3   | Generalization question | To what extent can the imperviousness of surfaces, for which imperviousness cannot be extracted from the BGT data set, be predicted with Computer Vision techniques? |

Table 1: Overview of sub-questions incl. label to facilitate referencing in this report

 # Formalization 

The research question at hand is formulated using the terminology commonly employed in the field of image recognition. This approach is adopted to minimize potential ambiguity and ensure transparency in the interpretation of the question.

In our research, the question is approached as a semi-supervised, binary image segmentation problem with hard labels. Let's first delve into the concept of image segmentation, which is the process assigning a label to every pixel in an image. This assignment of a label is performed in a semi-supervised way, which refers to the availability of ground truth labels for these pixels. As the previously defined "Generalization question" implies, we do not have ground truth labels for all pixels. Consequently, employing a fully supervised learning approach is not feasible. Nevertheless, the "Reproduction question" indicates that some ground truth labels can be extracted. Consequently, we opt for a semi-supervised approach that leverages the available information.

Next, we consider the distinction between binary and multi-class classification. Given that this is solely a pilot project, we choose to tackle a binary classification problem, distinguishing between impervious and non-impervious surfaces. The rationale behind this decision is to commence with the simplest form of classification and increase complexity if the initial results prove unsatisfactory.

Furthermore, we need to address the differentiation between hard and soft labels. For this study, we decide to solely utilize surfaces from the BGT dataset, where it can be assumed that the entire surface is either impervious or non-impervious. Consequently, we employ hard labels, which indicate an absolute classification without considering any degree of uncertainty. An alternative approach would involve incorporating partially impervious surfaces with soft labels that reflect the probability of being impervious or non-impervious. However, due to the need for additional assumptions (such as the average proportion of imperviousness in backyards), we opt against this approach in favor of employing hard labels.

# Methodology

The methodology can be described as a five-step process as shown in Figure 2. It is based on a standardized data science framework called CRISP-DM (Shearer, 2000) and tailored to the research question in scope. The following paragraphs will describe each of the five steps of the methodology, how they relate to the CRISP-DM framework and the rationale for tailoring to the research question if so. It is important to note that the first part of the CRISP-DM framework, which is called “Problem understanding”, is not covered by the five-step approach since this is covered by sections 4.1 Research question and 4.2 Formalization.

Figure 2: High level overview of the methodology incl. mapping to questions and important notes

## Surface extraction

The BGT (Dutch acronym for Basisregistratie Grootschalige Topografie) is a dataset that contains geometries and description of what these geometries represent in the real world. Based on these geometry descriptions, assumptions are made whether these geometries represent impervious or non-impervious surfaces in the real world.

Imperviousness is defined as “covering of the soil surface with impermeable materials because of urban development and infrastructure construction” in this pilot. It is based on the definition used by the European Environment Agency (EEA, 2022). Three properties of the geometries in the BGT are considered for extraction: 1) geometry type, 2) geometry category and 3) physical appearance. Surfaces that completely adhere to the definition of impervious or non-impervious are extracted.

It is important to note that the extraction of surfaces from the BGT that adhere to the definition of impervious or non-impervious is done without involvement of domain experts in impervious surfaces. Although the lack of domain expertise is a major limitation to this pilot, the methodology is still valid for the purpose of the research question in scope.

The output of the surface extraction-step is a set of geometries labelled as “completely impervious” and another set labelled as “completely non-impervious”. For example, all geometries of type “surface” and category “building” are in the set labelled as “completely impervious”. All geometries of type “surface”, category “bare area of land” and physical appearance “sand” are in the set labelled as “completely non-impervious”. The complete overview of extracted surfaces is given in Appendix 5.

The surface extraction-step is not part of the CRISP-DM framework but added as tailoring for this research question. Although this step could have been included as sub-process under “data preparation”, it is marked as a standalone step to clearly mark the assumptions made. These assumptions form the basis for the succeeding steps and should be clearly marked.

## Data preparation

The data preparation step explains how data is organized for modeling. It is done based on three concepts: 1) Image files, 2) BGT objects and 3) Tiles (see Figure 3). The process of preparing data is performed twice, once for satellite imagery and once for aerial imagery. The data preparation step is part of the CRISP-DM framework, so it’s common practice for a data science project.

### Image files

For the satellite imagery, a single image file with a mosaic of two different orbits of the SuperView satellite is used as a source. The image file is obtained via a website hosted by the Netherlands Space Office called “Satellietdataportaal” (Dutch for “Satellite data portal”). The exact date of the two orbits is not explicitly mentioned, but based on the information provided on the portal, they seem to be dated 6 May and 8 May 2022. The spatial resolution is 50 cm and it contains three bands (red, green and blue) with a value between 0 and 255 (8 bits) for each pixel and each band.

For the aerial imagery, 256 non overlapping image files are used as source. The image files are obtained from a website hosted by “Beeldmateriaal Nederland”. The 1km by 1km tiles from a layer called “Kaartbladen 2022” are downloaded. The images are captured between February and April 2022. The spatial resolution is 7.5 cm and it contains three bands (red, green and blue) with a value between 0 and 255 (8 bits) for each pixel and each band.

Figure 3: Overview of data preparation steps

### BGT objects

The BGT dataset serves as a source of ground truth labels. An API provided by is utilized to obtain objects including attributes such as feature type, geometry type, and the date of creation. Duplicate entries are identified based on the “identificatie.lokaalID” attribute, the object with the most recent date is retained (all others are dropped).

### Tiles

In computer vision algorithms, a tile is the fundamental format for processing data. In this study, the tiles are derived from the raster of the image files and the geometry of the specific region under examination, which in this case is the municipality of Utrecht.

To initiate the tiling process, the image file containing the top-left point of the envelope of the region in scope is identified as the "reference pixel." This reference pixel serves as a starting point to generate a rectangular grid of non-overlapping tiles covering the entire envelope of Utrecht. Each tile has user-defined dimensions of 256 by 256 pixels, to conform with the input format of the computer vision algorithm.

From the resulting rectangular grid of tiles, only those tiles that cover or mask the region in scope are selected. These tiles form the backbone of the dataset, i.e. they define the format in which input data will be presented to the computer vision model.

### Input data

The input data consists of RGB values and corresponding ground truth labels. The input data is formatted using tiles, as described in the previous section. To construct the input data, RGB values and ground truth labels are extracted from their source. RGB values are copied from the image file corresponding to each tile.

Obtaining the ground truth labels involves rasterizing the BGT objects according to the tile format. Each pixel in the raster is assigned a ground truth label based on the corresponding geometry. If a pixel is covered by a geometry labeled as "completely impervious", the ground truth label is set to 0. Similarly, if a pixel is covered by a geometry labeled as "completely non-impervious", the ground truth label is set to 1. If a pixel is covered by both types of geometry or no geometry at all, the ground truth label is set to 2, representing the label "unknown".

Once the RGB values and ground truth labels are integrated into the tiles format, the data is cleansed. Section 2.4 describes the details and rationale for cleansing steps. It is important to note here that tiles that are completely or to a large extent covered with clouds are excluded from the dataset. Clouds obstruct the view on the surface area and therefore do not contribute to the aim of this study. Moreover, not removing tiles with cloud coverage is expected to have a negative impact on the training process.

To ensure fast reading of the input data during training and inference, the RGB values and ground truth labels are stored in the HDF5 format on disk.

## Modelling

The modelling step is part of the CRISP-DM framework and explains which model architecture is selected, on which data the model is trained, which algorithm is used for training, with which so called hyperparameters the training algorithm was executed and how the final model is selected.

Model architecture

The model architecture employed is a U-net, a specialized neural network designed for image segmentation tasks. Image segmentation aligns with the formalization of the research question addressed in this study.

The selection of the U-net architecture is based on its demonstrated ability to achieve state-of-the-art results (David et al., 2022). Its design consists of a contracting path (aka encoder) for capturing context and an expansive path (aka decoder) for precise localization (Ronneberger et al., 2015).

To expedite the training process and leverage previous advancements, the U-net model is initialized with pre-trained weights. Previous studies have shown that utilizing pre-trained weights can yield state-of-the-art results even with limited training time (Iglovicov et al., 2018). These pre-trained weights are obtained from an image classification task conducted on a benchmark dataset known as ImageNet.

## Data splitting

In accordance with common machine learning practices (Gareth et al., 2013), the entire dataset is divided into a training, validation and test set. The split is performed randomly at the tile level, ensuring that each subset contains representative samples.

The train set is used to train the model. The validation set allows for monitoring the model's performance during training. Finally, the test set serves as an evaluation set to assess the model's generalization ability on unseen data.

The dataset is split with a ratio of 80% for training, 10% for validation, and 10% for testing. This distribution is chosen based on the assumption that an 80% training set is sufficiently large to effectively train the model without overfitting.

## Model training

The model training process is divided into three distinct phases, with each phase focusing on training specific parts of the model (see Figure 4).

In the first phase, only the classification head of the model is trained. By isolating this part of the network, the model can learn to classify and differentiate between different classes based on the input data.

In the second phase, both the classification head and the decoder are trained. This allows the model to improve its ability to generate segmentation maps based on the classification information.

The third and final phase involves training the entire model, including the classification head, decoder, and encoder. This comprehensive training ensures that all components of the model are fine-tuned and optimized together.

Figure 4: Timeline of three-phases training process for aerial imagery

During the training process, a weighted cross-entropy loss function is employed to measure the discrepancy between predicted and ground truth labels. The weights of the cross-entropy loss are user defined parameters that do not change during training, so called hyperparameters. These weights are set with two objectives: 1) The results should be useful for answering the generalization question of this study and 2) the recall for the classes "completely impervious" and "completely non-impervious" are similar. In order to meet the first objective, the weight for a false classification of the class “unknown” is set to 0. This gives the model no “incentive” to predict the “unknown” class, so either "completely impervious" or "completely non-impervious" can be expected as prediction. To meet the second objective, similar recall for both "completely impervious" or "completely non-impervious", the ratio of the weights of these classes is set to the inverse of the ratio of occurrence in the data set.

The Adam optimizer is utilized to update the model's parameters and optimize the loss function. Adam combines adaptive learning rates and momentum, allowing for efficient convergence during the training process (Kingma et al., 2014).

To further optimize the training process, a learning rate scheduler is employed. This scheduler dynamically adjusts the learning rate based on the model's performance. Specifically, it reduces the learning rate when the model's improvement plateaus, enabling more precise fine-tuning and preventing overfitting (Bengio, 2012).

Evaluation of the model's performance is conducted using the F1 score over the classes "completely impervious" and "completely non-impervious". The F1 score provides a balanced measure of precision and recall.

## Model selection

The choice of the model architecture is left to the user, the values of the model parameters are learned and updated to optimize the model's performance.

To ensure backtracking capabilities and to monitor the progress of the model, the values of the parameters are saved to disk at the end of each epoch. This allows for easy retrieval of previous versions of the model if needed.

During the inference phase, the user selects the desired version of the model based on specific requirements or preferences. In this study, the model with the highest F1 score for the classes "completely impervious" and "completely non-impervious" over the validation set is chosen as the final model for inference.

## Macro evaluation

In this study, the evaluation step in the CRISP-DM framework is divided into two distinct steps: Macro evaluation and Micro evaluation. This division is essential because not all pixels in the dataset can be evaluated using the same approach. When the ground truth label is labelled as "unknown," evaluating those pixels based on the ground truth label becomes impractical. However, these pixels can still be assessed through visual inspection conducted by a human. It is crucial to understand that visual inspection demands more manual effort compared to the computational evaluation using the ground truth label. As a result, the extent to which human inspection can be performed is limited by the project's available resources. This limitation prompts the introduction of the terms "macro evaluation" and "micro evaluation."

Macro evaluation entails evaluating the entire test set on a macro scale and is performed computationally with a quantitative approach. It involves computing the confusion matrix for the test set and subregions within it, as well as calculating the recall for each individual tile.

## Micro evaluation

Micro evaluation involves evaluating randomly sampled tiles within the test set on a micro scale. This evaluation is done manually and takes a qualitative approach. It requires visually inspecting tiles to identify illustrative examples that reflect the quality of reproduction and generalization. It is important to note that micro evaluation is a best-effort approach, and there is no guarantee of representativeness in the selected tiles for inspection.

# Exploratory data analysis

The data used in this study is explored before and after the preparation step. The data exploration before preparation is done mainly to understand which preparation steps are necessary or desirable. The data exploration after preparation is done to mainly to make informed decisions to configure the training algorithm.

## Before preparation

As mentioned before, the BGT dataset serves as a source of ground truth labels. The BGT contains objects including attributes such as feature type and geometry type. To provide insight into the objects in the dataset, a breakdown is given in Figure 5. It’s important to note that this figure describes only the set of objects that resulted from the surface extraction step.

Figure 5: Breakdown of BGT objects by type

The image files for both satellite and aerial are explored by analysing the following image characteristics on a tile level: entropy, peak-signal-to-noise-ratio, average difference, average edge, average gradient and contrast. The analysis reveals that contrast is a useful metric for detecting cloud coverage, as shown in Figure 6.

Figure 6: Satellite imagery and it’s contrast on tile level

The analysis on contrast per tile resulted in an exclusion of XXX tiles for the satellite imagery data set due to cloud coverage. Figure 7 shows the location of the excluded tiles. The contrast analysis was also conducted on aerial imagery but did not result in any tile exclusions.

Figure 7: Tiles excluded from satellite data set

## After preparation

The data exploration after preparation is done mainly to make informed decisions to configure the training algorithm. The main insight after exploration is based on Figures 8 and 9.

Figure 8: breakdown of the ground truth labels per subregion for the satellite imagery dataset

Figure 8 shows the breakdown of the ground truth labels per subregion in Utrecht for the satellite imagery dataset. Overall, the distribution of ground truth labels is 3:5:2 for “completely impervious”, “completely non-impervious” and “unknown” respectively. It is important to note that the distributions differ per subregion. For example, the share of “completely impervious” is twice as large for “Binnenstad” as the overall dataset. Also, the share of “completely non-impervious” is more dominant for “Vleuten-De Meern” than in the overall dataset. These are important observations for configuration of the training algorithm and evaluation of the results.

Figure 9: breakdown of the ground truth labels per subregion for the aerial imagery dataset

Another notable observation from Figure 8 is the total pixel count of approximately 400 million in the satellite imagery dataset, giving a sense of the dataset's order of magnitude. Figure 9 provides the same chart for aerial imagery, showing the breakdown of label shares similar to satellite imagery. However, the total number of pixels in the aerial imagery dataset is approximately 16 billion, which is 40 times larger than the satellite imagery dataset. This difference can be attributed to the higher resolution of aerial imagery, as the number of pixels scales with the square of the resolution ratio (50 cm / 7.5 cm)² ≈ 44.

## Modeling

In section 2.3, the three phases of training for the models are discussed: 1) classification head only, 2) classification head and the decoder, and 3) the entire model. The progress of training for aerial and satellite imagery can be observed in Figure 4 and Figure 10, respectively.

Figures 4 and 10 reveal that both the models exhibit the most significant improvement during the initial epoch of phase 2. During this epoch, the loss decreases, and the f1 score experiences the greatest increase. However, as training progresses, the performance enhancements become more gradual and a tendency to slightly overfit becomes apparent. This overfitting becomes evident when comparing the loss and f1 score between the training and validation sets (see Appendix 7).

Figure 10: Timeline of three-phases training process for satellite imagery

The final models are chosen based on the best F1 score achieved on the validation set. The confusion matrices for the final models in aerial imagery and satellite imagery are presented in Figures 11 and 12, respectively.

Figure 11: Confusion matrices for aerial imagery

Upon analysis of the confusion matrices, several observations can be made. Firstly, the models do not predict the "Unknown" class, which aligns with their intended design. This indicates that the models either predict a pixel as "completely impervious" or "completely non-impervious" when the ground truth label is "unknown". This data provides valuable insights for analysing the generalization aspect of this study.

Secondly, the recall values for both classes are similar, which is also by design. This indicates that the models perform consistently well in terms of identifying both impervious and non-impervious classes.

Figure 12: Confusion matrices for satellite imagery

Lastly, there is a slight indication of overfitting, as the recall values for both classes are slightly lower on the validation set compared to the training set. However, for the purpose of this study, this slight overfitting is not deemed problematic. The primary objective is to investigate the reproduction and generalization capabilities of computer vision algorithms, rather than training a model specifically for production purposes.

# Evaluation

As explained in the Section 2.3, the evaluation is performed on a macro and micro level. Macro evaluation entails evaluating the entire test set on a macro scale and is performed computationally with a quantitative approach. Micro evaluation involves evaluating randomly sampled tiles within the test by human visual inspection and takes a qualitative approach.

Macro evaluation

The test set serves as an evaluation set to assess the model's generalization ability on unseen data. Figures 13 and 14 show the confusion matrix over the test set for aerial and satellite imagery respectively. Examining Figure 13, which focuses on aerial imagery, it can be observed that the recall rates for both "completely impervious" and "completely non-impervious" are 96%. These recall values closely match those obtained from the validation set, as depicted in Figure 11. Hence, the model demonstrates a consistent ability to generalize well to new data that share similar characteristics with the test set.

The test set can also be used to benchmark our results, for which we turn to (Sobieraj et al., 2022). Their study reported recall values of 98% for non-impervious areas (a +2% deviation from our results) and 94% for impervious regions (a -2% deviation from our results). While these percentages offer a basic point of comparison, it's essential to acknowledge the disparities in methodology and scope. For example, (Sobieraj et al., 2022) used their entire dataset for model training, a departure from our approach that involved splitting the data into training, validation, and test sets. Additionally, their research focused on a relatively compact residential estate complex constructed between 1999 and 2011, covering just one square kilometre. In contrast, our study examined the expansive municipality of Utrecht, spanning centuries of construction history and encompassing an area approximately 100 times larger. So our recall values are considered on-par with a study published in 2022, the contrast in scope and methodology should be considered when discussing the true implications of this benchmark.

Figure 13: Confusion matrix on test set for aerial imagery

Moving on to Figure 14, showcasing the confusion matrix for satellite imagery, we note that the recall rates for both "completely impervious" and "completely non-impervious" are 90%. These recall values align closely with those derived from the validation set, as illustrated in Figure 12. Consequently, the model showcases robust generalization capabilities towards unseen data possessing characteristics similar to the test set.

Figure 14: Confusion matrix on test set for satellite imagery

The obtained results suggest that the model's recall can be anticipated to be similar when applied to previously unobserved data sharing comparable characteristics. However, a cursory examination of relevant literature reveals arguments that cast doubt on the generalizability of the model's performance across aerial or satellite imagery captured in different geolocations or under varying conditions. For instance, (Lunga et al., 2021) highlights challenges in spatio-temporal generalization for satellite imagery, such as variations in environmental factors and image scenes (such as changing geographies, vegetation types, different spatial arrangements in inhabited areas, etc.). Similarly, (Nalepa et al., 2021) emphasizes the importance of simulating atmospheric conditions to enable learners that exhibit strong generalization capabilities across image data acquired under diverse imaging settings. These findings underscore the need for further investigation, for example if the stratification approach presented by (Lunga et al., 2021) or simulation approach presented by (Nalepa et al., 2021) can mitigate the generalizability problem.

Figure 15: Confusion matrices over test set for aerial imagery for two sub-regions: “Vleuten–De Meern” (sub-urb) and “Binnenstad” (city centre)

In order to examine the spatial consistency of performance, a comparative analysis was conducted on two regions characterized by extreme imperviousness profiles. The first region, "Vleuten-De Meern," represents a suburb with the highest proportion of "completely non-impervious" areas, while the second region, the city centre of Utrecht known as "Binnenstad," exhibits the largest share of "completely impervious" regions (refer to Figure 8 and 9 for imperviousness profiles). Figures 15 and 16 display the comparison results for aerial and satellite imagery, respectively.

Figure 16: Confusion matrices over test set for satellite imagery for two sub-regions: “Vleuten–De Meern” (sub-urb) and “Binnenstad” (city centre)

Both Figures 15and 16 reveal recall percentages that deviate from those observed over the entire test set (Figures 13 and 14). This disparity can be attributed to the class distribution within the dataset and the distribution-based loss function employed. For instance, in "Vleuten-De Meern," the ratio of pixels labelled as "completely non-impervious" to "completely impervious" is 71:15, while the loss function assigns weights based on a ratio of 50:31. Similarly, in "Binnenstad," the ratio of pixels labelled as "completely non-impervious" to "completely impervious" is 13:60, with the loss function weights set at 50:31. In both cases, the model does not penalize errors in the dominant class sufficiently, resulting in lower recall rates for the minority class.

This problem has also been addressed in the literature. For example, a study (Hongzhang et al., 2023) found that region-based loss functions perform better than distribution-based loss functions in segmenting dense road networks in urban areas. Therefore, employing a region-based loss function like Jaccard might improve consistency in the recall percentages.

Additionally, the addition of an attention mechanism (Oktai et al., 2018) to the model architecture might further enhance the results. The attention mechanism can help the model focus on relevant features and improve its ability to capture important details, leading to more accurate segmentation outcomes.

Figures 17, 18, 19 and 20 give a more detailed overview of the spatial distribution of recall.

Figure 17: Spatial distribution of recall “completely non-impervious” for aerial imagery

Figure 18: Spatial distribution of recall “completely impervious” for aerial imagery

Figure 19: Spatial distribution of recall “completely non-impervious” for satellite imagery

Figure 20: Spatial distribution of recall “completely impervious” for satellite imagery

## Micro evaluation

Micro evaluation involves evaluating randomly sampled tiles within the test set on a micro scale, i.e. by visually inspecting tiles to identify illustrative examples that reflect the quality of reproduction and generalization.

Figure 21 is an example of a tile for which generalization over the pixel labelled "unknown" is considered good. These pixels are correctly classified as either "completely impervious" or "completely non-impervious," and the edges between the road and verge align closely with how a human would have drawn them. Additional examples of scenes with roads demonstrating good generalization can be found in Figures 41 to 50 in Appendix 6. Our hypothesis would be that favourable generalization can be attributed to a large number of completely labelled scenes with roads. The algorithm has been exposed to an adequate number of examples, allowing it to learn how to accurately label such scenes.

Figure 21: Example of good generalization in road (surroundings) context for aerial imagery

Figure 22 illustrates poor generalization over backyards. In the left-hand side of the image, two distinct backyards can be seen. The upper left backyard features a lawn with a path of tiles and a trampoline, while the lower left backyard appears to be predominantly covered with impervious surfaces. The predicted labels for the upper left backyard are predominantly "completely non-impervious," which is incorrect for the path of tiles. The classification of the trampoline is hard to qualify, since the correct label could be ambiguous for human annotators. In the lower left backyard, the classification is a mix of "completely impervious" and "completely non-impervious," which seems incorrect as it appears mostly "completely impervious."

More examples of scenes with backyards demonstrating poor generalization can be found in Figures 51 to 59 in Appendix 6. Our hypothesis suggests that the lack of labelled backyards is a contributing factor. The algorithm has not been exposed to a sufficient number of examples to learn how to accurately label such scenes, hence leading to poor generalization in this context.

Figure 22: Example of poor generalization over backyards

Figure 23 shows a rare but insightful example of a surface without vegetation being classified as "non-impervious". This example is worth highlighting because it presents a potential benefit compared to imperviousness assessments based on vegetation indices. The use of vegetation indices as a tool to estimate urban impervious surface fractions relies on the assumption of an inverse relationship between vegetation cover and impervious surface. It is implicitly assumed that non-impervious surfaces within urban areas are covered with green vegetation (Kaspersen, 2015). However, this assumption is not always true, as demonstrated by surfaces like bare soil, which are non-impervious but lack vegetation. This example illustrates a potential benefit of the methodology of this study compared to vegetation indices based methods.

Figure 23: Example of a surface without vegetation being classified as "non-impervious"

Figure 24 is an example of a tile from the satellite dataset for which generalization over the pixel labelled "unknown" is considered good. These pixels are correctly classified as "completely impervious". However, the reproduction of the paths labelled “completely impervious” is poor, which can be explained by the obstruction of trees.

Figure 24: Example of good generalization in industrial context for satellite imagery

Figure 25 is an example of a tile from the satellite dataset for which generalization over the pixel labelled "unknown" is considered poor. The backyards are predominantly labelled “completely impervious”, and although it's hard to visually inspect whether the predictions are correct, it seems unlikely that such large extents of the backyard are indeed “completely impervious”.

Figure 25: Example of poor generalization in residential context for satellite imagery

# Conclusion

The growth of impervious surfaces in cities exposes them to increased vulnerability from extreme rainfall and high temperatures. Previous research has demonstrated the capability of computer vision algorithms to classify pixels in aerial images and satellite photos into impervious and non-impervious categories. However, it remains uncertain to what extent computer vision is a suitable technique for developing indicators at PBL (Netherlands Environmental Assessment Agency). This raises the question to what extent image recognition algorithms can identify impervious surfaces in urban areas using satellite images/aerial photos, and impervious surfaces extracted from the BGT dataset.

The extraction question addresses the ability to extract impervious and non-impervious surfaces from the BGT dataset. It emphasizes the importance of input from domain experts, as their expertise in imperviousness and the BGT dataset is crucial. The mapping from BGT objects to class labels forms the foundation for subsequent steps and should be based on clear class definitions. This mapping process should consider attributes in the BGT data, as well as visual inspection of the imagery data to ensure conformance to the class definitions.

The reproduction question investigates the extent to which image recognition algorithms can reproduce impervious and non-impervious surfaces extracted from the BGT dataset. Macro evaluations of this study have shown that the model achieved an average recall of 90% and 96% for ground truth labels based on 50 cm resolution satellite imagery and 7.5 cm resolution aerial imagery, respectively. Overall, our recall values are considered on-par with a similar study published recently (2022). However, spatial consistency in recall is lacking, with lower recall rates observed for the "completely non-impervious" class in the city centre. This discrepancy can be attributed to the class distribution in the dataset and the design of the loss function. Mitigation measures, such as deploying a region-based loss function or adding an attention mechanism to the neural network, might help to address this issue.

The generalization question focuses on the ability of image recognition algorithms to predict imperviousness for surfaces where imperviousness cannot be extracted from the BGT dataset. Visual inspections reveal that the results generalize well to images with roads but perform poorly in backyard areas. This issue has also been highlighted in the literature as a generalization challenge. Mitigation measures are being explored to overcome this limitation, including approaches such as a comprehensive stratification approach or simulation of atmospheric conditions during model training.

In conclusion, answering the overarching research question regarding the identification of impervious surfaces in urban areas using satellite images, aerial photos, and the BGT dataset is a complex task. While advancements in computer algorithms have provided powerful image segmentation models, achieving consistent and expressive results remains a challenge. The extraction of ground truth labels requires a multidisciplinary approach, as it plays a crucial role in determining the expressiveness of the end results. Additionally, ensuring consistency within seen data and generalizability to unseen data are hurdles that need to be addressed before usage for PBL publications.