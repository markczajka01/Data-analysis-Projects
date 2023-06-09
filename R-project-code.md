# Data-analysis-Project-R-
Performance Modelling code in R

library("dplyr")
library("sjPlot")
library("leaps")
library("ggfortify")
library("Hmisc")
library("qtlcharts")
library("vtable")
library("caret")
library("tidyverse")
library("olsrr")
library("plotly")


# load data
filename = "student-por.csv"
d2 = read.table(filename, sep=";", header=TRUE)
#d2 = subset(d2, select = -c(G1,G2) )


# Correct Data Classification into factors:
cols = c("famrel", "freetime", "goout", "Dalc", "Walc", "health",
         "traveltime","studytime", "Medu", "Fedu")
d2[cols] <- lapply(d2[cols], as.factor)


# Summaries of numeric variables:

num_d2 = d2 %>%
  dplyr::select(where(is.numeric))

sumtable(num_d2,
         summ=c('notNA(x)',
                'mean(x)',
                'median(x)',
                'propNA(x)'))
                
                
                
# Summaries of factor variables:

fact_d2 = d2 %>%
  dplyr::select(where(is.factor))
sumtable(fact_d2,
         summ=c('notNA(x)',
                'mean(x)'))
                
# Summaries of character variables:

char_d2 = d2 %>%
  dplyr::select(where(is.character))

st(char_d2,
         summ=c('notNA(x)',
                'mean(x)'))
              
# Understanding G3 

g1 = d2 |> ggplot() + aes(x = 0, y = G3) +
                 geom_violin(trim=TRUE) + 
                 stat_summary(fun.data=mean_sdl, geom="pointrange", color="red") +
                 geom_jitter(aes(x=0, y=G3), alpha=0.25) +
                 ggtitle("Portuguese Final Exam Scores") +
                 ylab("Final Exam Score") +
                 xlab("")

ggplotly(g1)



# Correlation matrix (EXCLUSIVE Numeric Variables)

num_d2 = d2 %>%
  dplyr::select(where(is.numeric))

cor_mat = cor(num_d2)
melted_cor_mat = cor_mat %>%
  data.frame() %>% 
  rownames_to_column(var = "var1") %>% 
  gather(key = "var2", value = "cor", -var1)

ggplot(data = melted_cor_mat, 
       aes(x=var1, y=var2, fill=cor)) + 
  geom_tile() + theme_minimal(base_size = 30) +
  scale_fill_gradient2(
    low = "blue", high = "red", mid = "white", 
    midpoint = 0, limit = c(-1,1)) +
  theme(axis.text.x = element_text(angle = 90, hjust = 1))

qtlcharts::iplotCorr(num_d2) 



# Impact of ZERO scoring Students on the assumptions on the model?

model_all = lm(G3 ~ ., data = d2)
#summary(model_all)

# Add fitted and residual values to the data set

d2_resid_test = d2 %>%
  mutate(model_fit = model_all$fitted.values,
         model_resid = model_all$residuals)

# Add innovative plots - Homoscedasticity Test

ggplot(d2_resid_test, aes(x = model_fit, y = model_resid)) +
  geom_point() +
  theme_test() +
  labs(x = "Fitted", y = "Residual") +
  geom_hline(yintercept = 0) +
  geom_smooth(method = "loess", se = FALSE)


# Add more interesting plot - qq plot

ggplot(d2_resid_test, aes(sample = model_resid)) +
  geom_qq() +
  geom_qq_line() +
  theme_test() +
  labs(x = "Theoretical Quantiles", y = "Sample Quantiles")
  
  
  # Model assumptions without the ZERO scoring students:

d2_nozero = d2 %>%
  filter(G3 > 0) %>%
  filter(row_number() != 172 & row_number() != 62) 

# Checking for Assumptions for no failure model:

model_all = lm(G3 ~ ., data = d2_nozero)
#summary(model_all)

# Add fitted and residual values to the data set

d2_resid_test = d2_nozero %>%
  mutate(model_fit = model_all$fitted.values,
         model_resid = model_all$residuals)

# Add innovative plots - Homoscedasticity Test

ggplot(d2_resid_test, aes(x = model_fit, y = model_resid)) +
  geom_point() +
  theme_test() +
  labs(x = "Fitted", y = "Residual") +
  geom_hline(yintercept = 0) +
  geom_smooth(method = "loess", se = FALSE)


# Add more interesting plot - qq plot

ggplot(d2_resid_test, aes(sample = model_resid)) +
  geom_qq() +
  geom_qq_line() +
  theme_test() +
  labs(x = "Theoretical Quantiles", y = "Sample Quantiles")

plot(model_all, 4)
plot(model_all, 5)


M0 = lm(G3 ~ 1, data = d2_nozero)  # Null model
M1 = lm(G3 ~ ., data = d2_nozero)  # Full model

res = bind_rows(broom::glance(M0), 
                broom::glance(M1))
res$model= c("M0","M1")
res %>% pivot_longer(
  cols = -model, 
  names_to = "metric", 
  values_to = "value") %>% 
  pivot_wider(
    names_from = "model") %>% 
  gt::gt() %>% 
  gt::fmt_number(columns = 2:3, 
                 decimals = 2) %>% 
  gt::sub_missing()
  
  # AIC Models:

fw_aic = step(M0,
                     scope = list(lower = M0, upper = M1),
                     direction = "forward",
                     trace = FALSE)

bw_aic = step(M1,
                     direction = "backward",
                     trace = FALSE)

#AIC(forward_aic)
#AIC(back_aic)



# BIC Models:

fw_bic = step(M0,
                     scope = list(lower = M0, upper = M1),
                     direction = "forward",
                     k = log(nrow(d2)),
                     trace = FALSE)

bw_bic = step(M1,
                     direction = "backward",
                     k = log(nrow(d2)),
                     trace = FALSE)

#BIC(forward_bic)
#BIC(backward_aic)


# Creating a table for the summary of the models
sjPlot::tab_model(fw_aic, bw_aic, fw_bic, bw_bic, show.ci = FALSE, show.aic = TRUE,
  dv.labels = c("Forward model AIC", "Backward model AIC", "Forward model BIC", "Backward model BIC"))
  
 # --------------- Evaluate models----------------------

# Returns rmse and mae from 10 fold cross validation
# *for linear models on 'd2_nozero' df

cv0 <- function(formula, model = d2_nozero) {
    set.seed(4)
    train(formula, 
          model,
          method = "lm",
          trControl = trainControl(method = "cv", 
                                   number = 10),
          )
}
cv <- function(model) cv0(model$call$formula)

null_data = data.frame(median(d2_nozero$G3), G3 = d2_nozero$G3)

# Cross fold evaluate models
results = list(
    "Null" = cv0(G3 ~ ., null_data),
    "Full" = cv0(G3 ~ .),
    "Step forward AIC" = cv(fw_aic),
    "Step backward AIC" = cv(bw_aic),
    "Step forward BIC" = cv(fw_bic),
    "Step backward BIC" = cv(bw_bic))

# Plot RMSE values
resamples(results) |>
    ggplot(metric = "RMSE") +
    labs(y = "Root mean square error") +
    theme_minimal()

# Plot R-squared values
resamples(results) |>
    ggplot(results, metric = "Rsquared") +
    labs(y = "R² value") +
    scale_y_continuous(breaks = seq(0.8, 1, 0.02)) +
    theme_minimal()
    
    
    
    
 # Checking Assumptions on the final model (FOR BIC FORWARD ONLY):

# Add fitted and residual values to the data set

FM_resid_test = d2_nozero %>%
  mutate(FM_fit = fw_bic$fitted.values,
         FM_resid = fw_bic$residuals)

# Add innovative plots - Homoskedacitiy Test

ggplot(FM_resid_test, aes(x = FM_fit, y = FM_resid)) +
  geom_point() +
  theme_test() +
  labs(x = "Fitted", y = "Residual") +
  geom_hline(yintercept = 0) +
  geom_smooth(method = "loess", se = FALSE)


# Add more interesting plot - qq plot

ggplot(FM_resid_test, aes(sample = FM_resid)) +
  geom_qq() +
  geom_qq_line() +
  theme_test() +
  labs(x = "Theoretical Quantiles", y = "Sample Quantiles")

plot(model_all, 4)
plot(model_all, 5)




step.fwd.mod = ols_step_forward_aic(M1)
plot(step.fwd.mod) 
#%>%
  #ggplotly()

step.fwd.mod$model


step.back.mod = ols_step_backward_aic(M1)
plot(step.back.mod) 
#%>%
  #ggplotly()

step.back.mod$model
