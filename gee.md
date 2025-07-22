//ѡ����Ҫ�ü���ʸ������ 
var cc = ee.FeatureCollection("projects/ee-935239274z/assets/ayakekumu");
//ȥ�ƺ��� 
function maskL8sr(image) {
  var cloudShadowBitMask = (1 << 3);
  var cloudsBitMask = (1 << 5);
  var qa = image.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                 .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}
//ѡ��դ�����ݼ� 
var cc2019 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
                  .filterDate('2023-05-01', '2023-10-30')
                  .map(maskL8sr)
                  .median();
//�������ָ��                  
var mndwi = cc2019.normalizedDifference(['SR_B3', 'SR_B6']).rename('MNDWI');//����MNDWI
var ndbi = cc2019.normalizedDifference(['SR_B6', 'SR_B5']).rename('NDBI');//����NDBI
var ndvi = cc2019.normalizedDifference(['SR_B5', 'SR_B4']).rename('NDVI');//����NDVI
cc2019=cc2019.addBands(ndvi).addBands(ndbi).addBands(mndwi)
// ʹ�����в�����Ϊ����
var classNames = building.merge(water).merge(vegetation).merge(other);
var bands = ['SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6', 'SR_B7','MNDWI','NDBI','NDVI'];
// ͨ��Ҫ�ؼ���Landsat-8��ѡȡ��������landcover���Ը�������
var training = cc2019.select(bands).sampleRegions({
  collection: classNames,
  properties: ['landcover'],
  scale: 30
})
//�������� 
var withRandom = training.randomColumn('random');//���������������
// ����һЩ���ݽ��в��ԣ��Ա���ģ�͹�����ϡ�
var split = 0.7; 
var trainingPartition = withRandom.filter(ee.Filter.lt('random', split));//ɸѡ70%��������Ϊѵ������
var testingPartition = withRandom.filter(ee.Filter.gte('random', split));//ɸѡ30%��������Ϊ��������
//���෽��ѡ��smileCart() randomForest() 
var classifier = ee.Classifier.smileRandomForest(100).train({
  features: trainingPartition,
  classProperty: 'landcover',
  inputProperties: bands
});
//��Landsat-8���з���
var class_img = cc2019.select(bands).classify(classifier);
//���ò����������࣬ȷ��Ҫ���к�����������ݼ��Լ�����
var test = testingPartition.classify(classifier);
//�����������
var confusionMatrix = test.errorMatrix('landcover', 'classification');
print('confusionMatrix',confusionMatrix);//�������ʾ��������
print('overall accuracy', confusionMatrix.accuracy());//�������ʾ���徫��
print('kappa accuracy', confusionMatrix.kappa());//�������ʾkappaֵ

Map.centerObject(cc)
Map.addLayer(cc);
Map.addLayer(class_img.clip(cc), {min: 1, max: 4, palette: ['orange', 'blue', 'green','yellow']});
//���ش���õ�Ӱ��
Export.image.toCloudStorage({
 image:class_img.clip(cc),
 description: 'Landsat2019',
 region:ROI,
 scale:30,
 maxPixels:1e11
})