Sys.setenv(TZ='UTC') # Fuso horario local

# carregar bibliotecas
library(ggplot2)      # visualizacao de dados
library(sf)           # leitura e manipulacao de dados espaciais
library(data.table)   # manipulacao de dados
library(read.dbc)     # leitura de bases relacionais em Microsoft Access
library(geobr)        # dados espaciais do brasil
library(pbapply)      # progress bar
library(readr)        # rapida leitura de dados 
library(tidyr)        # manipulacao de dados
library(stringr)      # operacoes em strings
library(lubridate)    # dados em data/horario
library(fasttime)     # rapido processamento deddados em data/horario
library(mapview)      # visualizacao interativa dos dados
library(RColorBrewer) # paleta de cores
library(extrafont)    # fontes de texto
# library(bit.64)       # lidar com numeros ee 64bits
library(knitr)
library(furrr)
library(purrr)
library(future.apply) # Aplicar funcoes em paralelo
library(h3jsr) # H3 grade hexagonal
library(dplyr)
library(Hmisc) # calcular quantis ponderados
library(osmdata) # Download de dados do OpenStreeteMaps (OSM)
library(opentripplanner) # Usar OTP de dentro do R: https://github.com/ITSLeeds/opentripplanner



# Cria data.frame com municipios do projeto
munis_df <- data.frame(code_muni = c(3300803,3304300,3302007,3302700,3302858,3305554,3305752,3301850,3302270,
                                     3304144,3300456, 3301702,3301900,3302502,3303203,3303302,3303500,3303609,
                                     3304557,3304904,3305109,2304400,3550308,4106902,3106200,2927408,1501402,
                                     5300108),
                       name_muni=c('cachoeiras de macacu','rio bonito','itaguai','marica','mesquita','seropedica',
                                   'tangua','guapimirim','japeri','queimados','belford roxo','duque de caxias',
                                   'itaborai','mage','nilopolis','niteroi','nova iguacu','paracambi',
                                   'rio de janeiro','sao goncalo','sao joao de meriti','fortaleza','sao paulo',
                                   'curitiba','belo horizonte','salvador','belem','distrito federal'),
                       abrev_state=c('RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ','RJ',
                                     'RJ','RJ','RJ','RJ','RJ','RJ','RJ','CE','SP','PR','MG','BA','PA','DF'),
                       rm=c('rmrj','rmrj','rmrj','rmrj','rmrj','rmrj','rmrj','rmrj','rmrj',
                            'rmrj','rmrj','rmrj','rmrj','rmrj','rmrj','rmrj','rmrj','rmrj',
                            'rmrj','rmrj','rmrj','rmf','rmsp','rmc','rmbh','rms','rmb','ride-df'))


#funcao para transformar data frame em sf
to_spatial <- function(df1, coordenada = c("lon", "lat")) {
  x <- st_as_sf(df1, coords = coordenada, crs = 4326)
}

#funcao para remover acentos
rm_accent <- function(str,pattern="all") {
  if(!is.character(str))
    str <- as.character(str)
  pattern <- unique(pattern)
  if(any(pattern=="Ç"))
    pattern[pattern=="Ç"] <- "ç"
  symbols <- c(
    acute = "áéíóúÁÉÍÓÚýÝ",
    grave = "àèìòùÀÈÌÒÙ",
    circunflex = "âêîôûÂÊÎÔÛ",
    tilde = "ãõÃÕñÑ",
    umlaut = "äëïöüÄËÏÖÜÿ",
    cedil = "çÇ"
  )
  nudeSymbols <- c(
    acute = "aeiouAEIOUyY",
    grave = "aeiouAEIOU",
    circunflex = "aeiouAEIOU",
    tilde = "aoAOnN",
    umlaut = "aeiouAEIOUy",
    cedil = "cC"
  )
  accentTypes <- c("´","`","^","~","¨","ç")
  if(any(c("all","al","a","todos","t","to","tod","todo")%in%pattern)) # opcao retirar todos
    return(chartr(paste(symbols, collapse=""), paste(nudeSymbols, collapse=""), str))
  for(i in which(accentTypes%in%pattern))
    str <- chartr(symbols[i],nudeSymbols[i], str)
  return(str)
}

#funcao para transformar objeto `sf` de pontos para coordenadas lon e lat
sfc_as_cols <- function(x, names = c("lon","lat")) {
  stopifnot(inherits(x,"sf") && inherits(sf::st_geometry(x),"sfc_POINT"))
  ret <- sf::st_coordinates(x)
  ret <- tibble::as_tibble(ret)
  stopifnot(length(names) == ncol(ret))
  x <- x[ , !names(x) %in% names]
  ret <- setNames(ret,names)
  ui <- dplyr::bind_cols(x,ret)
  st_set_geometry(ui, NULL)
}
