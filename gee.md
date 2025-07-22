//选择需要裁剪的矢量数据 
var cc = ee.FeatureCollection("projects/ee-935239274z/assets/ayakekumu");
//去云函数 
function maskL8sr(image) {
  var cloudShadowBitMask = (1 << 3);
  var cloudsBitMask = (1 << 5);
  var qa = image.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                 .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}
//选择栅格数据集 
var cc2019 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
                  .filterDate('2023-05-01', '2023-10-30')
                  .map(maskL8sr)
                  .median();
//定义光谱指数                  
var mndwi = cc2019.normalizedDifference(['SR_B3', 'SR_B6']).rename('MNDWI');//计算MNDWI
var ndbi = cc2019.normalizedDifference(['SR_B6', 'SR_B5']).rename('NDBI');//计算NDBI
var ndvi = cc2019.normalizedDifference(['SR_B5', 'SR_B4']).rename('NDVI');//计算NDVI
cc2019=cc2019.addBands(ndvi).addBands(ndbi).addBands(mndwi)
// 使用下列波段作为特征
var classNames = building.merge(water).merge(vegetation).merge(other);
var bands = ['SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6', 'SR_B7','MNDWI','NDBI','NDVI'];
// 通过要素集在Landsat-8中选取样本，把landcover属性赋予样本
var training = cc2019.select(bands).sampleRegions({
  collection: classNames,
  properties: ['landcover'],
  scale: 30
})
//精度评价 
var withRandom = training.randomColumn('random');//样本点随机的排列
// 保留一些数据进行测试，以避免模型过度拟合。
var split = 0.7; 
var trainingPartition = withRandom.filter(ee.Filter.lt('random', split));//筛选70%的样本作为训练样本
var testingPartition = withRandom.filter(ee.Filter.gte('random', split));//筛选30%的样本作为测试样本
//分类方法选择smileCart() randomForest() 
var classifier = ee.Classifier.smileRandomForest(100).train({
  features: trainingPartition,
  classProperty: 'landcover',
  inputProperties: bands
});
//对Landsat-8进行分类
var class_img = cc2019.select(bands).classify(classifier);
//运用测试样本分类，确定要进行函数运算的数据集以及函数
var test = testingPartition.classify(classifier);
//计算混淆矩阵
var confusionMatrix = test.errorMatrix('landcover', 'classification');
print('confusionMatrix',confusionMatrix);//面板上显示混淆矩阵
print('overall accuracy', confusionMatrix.accuracy());//面板上显示总体精度
print('kappa accuracy', confusionMatrix.kappa());//面板上显示kappa值

Map.centerObject(cc)
Map.addLayer(cc);
Map.addLayer(class_img.clip(cc), {min: 1, max: 4, palette: ['orange', 'blue', 'green','yellow']});
//下载处理好的影像
Export.image.toCloudStorage({
 image:class_img.clip(cc),
 description: 'Landsat2019',
 region:ROI,
 scale:30,
 maxPixels:1e11
})