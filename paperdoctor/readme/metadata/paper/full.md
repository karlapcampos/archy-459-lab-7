<!-- page 1 -->
Marks of the Aurignacian: sign repertoires on mobile

objects from Germany and France

2026-11-17

library(tidyverse)

── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ── ✔ dplyr 1.2.1 ✔ readr 2.2.0 ✔ forcats 1.0.1 ✔ stringr 1.6.0 ✔ ggplot2 4.0.4 ✔ tibble 3.3.1 ✔ lubridate 1.9.5 ✔ tidyr 1.3.2 ✔ purrr 1.2.2 ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ── ✖ dplyr::filter() masks stats::filter() ✖ dplyr::lag() masks stats::lag() ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

signbase <- read_csv("signbase.csv", show_col_types = FALSE) theme_set(theme_minimal(base_size = 13) + theme(plot.title = element_text(face = "bold"), legend.position = "top"))

Introduction The Aurignacian, about 43,000–33,000 years ago, is the ﬁrst Upper Palaeolithic technocomplex to spread across most of Europe. Its best-known objects come from Hohle Fels and Geissenklösterle, two caves in the Lone Valley of the Swabian Jura, where ivory ﬁgurines and ﬂutes were found in well-dated layers (Higham et al., 2012). The Hohle Fels female ﬁgurine, carved from mammoth ivory, is at least 35,000 years old (Conard, 2009).

Most Aurignacian marks, however, are not ﬁgurative. They are notches, lines, dots and crosses on tools, ornaments and stone blocks. Repeated sequences of such marks have been read as pos- sible notational or information-storage devices (McPherron et al., 2022). If they were shared con- ventions, their repertoires might diﬀer between regions, as personal ornaments do (Vanhaeren & d’Errico, 2006). Using SignBase (Dutkiewicz et al., 2020), I ask whether sign repertoires diﬀer between Germany and France.

1

<!-- page 2 -->
Methods SignBase v1.0 (Bentz et al., 2020) records 531 Aurignacian mobile objects from 65 sites in 13 coun- tries, with 31 binary columns for the presence of each sign type. Germany (n = 264) and France (n = 148) provide most records, so I restricted the analysis to these two countries and removed duplicate records. I calculated the proportion of objects carrying each sign type in each country and counted the objects from each site.

# Focus on the two best-documented countries and remove duplicate records signs_focus <- signbase |> filter(country %in% c("Germany", "France")) |> distinct(site_name, object_type, .keep_all = TRUE)

sign_freq <- signs_focus |> pivot_longer(line:other, names_to = "sign", values_to = "present") |> summarise(prop = mean(present, na.rm = TRUE), n_present = sum(present, na.rm = TRUE), .by = c(country, sign))

top10 <- sign_freq |> summarise(n = sum(n_present), .by = sign) |> slice_max(n, n = 10, with_ties = FALSE) |> pull(sign)

site_counts <- signs_focus |> count(country, site_name) |> mutate(share = n / sum(n), .by = country)

top_site <- site_counts |> slice_max(n, n = 1, by = country, with_ties = FALSE) top_de <- filter(top_site, country == "Germany") top_fr <- filter(top_site, country == "France")

Results The analysed sample contains 108 objects (46 German, 62 French). The most frequent sign type is notch in Germany and notch in France. The German objects come from 9 sites and the French objects from 21. Figure 1 shows how the common sign types are spread across these sites.

site_signs <- signs_focus |> pivot_longer(all_of(top10), names_to = "sign", values_to = "present") |> summarise(n = sum(present, na.rm = TRUE), .by = c(country, site_name, sign)) |> filter(n > 0) |> left_join(select(site_counts, site_name, n_site = n), by = "site_name")

site_signs |> ggplot(aes(factor(sign, levels = top10), fct_reorder(site_name, n_site), size = n)) +

2

<!-- page 3 -->
geom_point() + facet_grid(rows = vars(country), scales = "free_y", space = "free_y") + scale_size_area(max_size = 8) + labs(title = "Sign types by site", x = NULL, y = NULL, size = "Objects") + theme(axis.text.x = element_text(angle = 45, hjust = 1))

3

<!-- page 4 -->
Figure 1: Number of objects at each site carrying each of the ten most common sign types. Sites are ordered by their total number of analysed objects; sites with none of these sign types are

not shown.

4

<!-- page 5 -->
Discussion These patterns should be read cautiously. Vogelherd alone supplies 30% of the German objects, and Castanet supplies 13% of the French ones, so a national diﬀerence may reﬂect the excavation and preservation history of a few sites. Objects from the same layer are not independent, so sites rather than objects are the better unit of comparison.

Conclusion In this sample, Germany and France share their common sign types and diﬀer only in rare ones. Notches and lines are the two most frequent signs in both countries. Notches appear on 57% of German objects and 45% of French objects, and lines on 35% and 35%. The contrasts are in signs found on fewer than ten objects: oblique lines occur on 15% of German objects but 2% of French ones, and vulva signs appear only in France, on 8% of French objects. Three sites (Vogelherd, Hohle Fels, and Geissenklösterle) supply 80% of the German objects, so these few rare-sign contrasts do not yet show separate regional traditions.

References Bentz, C., Dutkiewicz, E., Russo, G., & Lee, S. (2020). SignBase main data ﬁle [Data set]. ﬁgshare. https://doi.org/10.6084/m9.ﬁgshare.12000018.v1

Conard, N. J. (2009). A female ﬁgurine from the basal Aurignacian of Hohle Fels Cave in south- western Germany. Science, 459(7244), 248–252.

Dutkiewicz, E., Russo, G., Lee, S., & Bentz, C. (2020). SignBase, a collection of geometric signs on mobile objects in the Paleolithic. Scientiﬁc Data, 7, 364. https://doi.org/10.1038/s41597-020-00704- x

Higham, T., Basell, L., Jacobi, R., Wood, R., Bronk Ramsey, C., & Conard, N. J. (2012). Testing models for the beginnings of the Aurignacian and the advent of ﬁgurative art and music: The radiocarbon chronology of Geißenklösterle. Journal of Human Evolution, 62(6), 664–676.

McPherron, S. P., Archer, W., Otárola-Castillo, E. R., Torquato, M. G., & Keevil, T. L. (2022). Ma- chine learning, bootstrapping, null models, and why we are still not 100% sure which bone surface modiﬁcations were made by crocodiles. Journal of Human Evolution, 164, 103071.

Vanhaeren, M., & d’Errico, F. (2006). Aurignacian ethno-linguistic geography of Europe revealed by personal ornaments. Journal of Archaeological Science, 33(8), 1105–1128.

5
