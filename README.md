# main-form.cs


using System;
using System.Data;
using System.Drawing;
using System.IO;
using System.Windows.Forms;
using Accord;
using System.Threading;
using Accord.Controls;
using Accord.IO;
using Accord.MachineLearning.VectorMachines;
using Accord.MachineLearning.VectorMachines.Learning;
using Accord.Math;
using Accord.Statistics.Analysis;
using Accord.Statistics.Kernels;
using AForge;
using AForge.Imaging;
using AForge.Imaging.Filters;
using AForge.Imaging.Textures;
using Accord.Imaging;
using Accord.MachineLearning;
using System.Text;
//using Components;
using ZedGraph;
using System.Collections.Generic;

using System.Drawing.Imaging;
using System.Collections;
using System.ComponentModel;
using System.Linq;
using System.Threading.Tasks;


namespace SVM
{
    
    public partial class MainForm : Form
    {
        paramClass myObject = new paramClass();
        List<string> asd = new List<string>();
        
        
        SupportVectorMachine svm;

        string[] columnNames; 

       
        String path;
        private System.Drawing.Bitmap Thresholded;
        List<string> alist = new List<string>();
        int count = 0;
        int i, itteration = 0;
        int filelength = 0;
        int times = 0;
        FileInfo file1 = null;


        
        int[] expected;
        int Gender;

        int lengthOfFeature = 0;
        double[] result= new double[20000];

        public MainForm()
        {
            InitializeComponent();

            dgvLearningSource.AutoGenerateColumns = true;
            dgvPerformance.AutoGenerateColumns = false;

            openFileDialog.InitialDirectory = Path.Combine(Application.StartupPath, "Resources");
        }

        private void ApplyFilter(IFilter filter)
        {

            System.Drawing.Bitmap filteredImage;


            
            filteredImage = filter.Apply(Grayscale.CommonAlgorithms.RMY.Apply((Bitmap)pictureBox1.Image));

           
            pictureBox1.Image = filteredImage;
        }

         
        private static System.Drawing.Image ResizeImage(System.Drawing.Image image, Size newSize)
        {
            System.Drawing.Image newImage = new Bitmap(newSize.Width, newSize.Height);
            using (Graphics GFX = Graphics.FromImage((Bitmap)newImage))
            {
                GFX.DrawImage(image, new Rectangle(System.Drawing.Point.Empty, newSize));
            }

            return newImage;

        }
        



        int[] list = new int[500];
        private  int[] UseParams()
        {

            for (int i = 0; i < list.Length; i++)
            {

                list[i] = i;

                
            }
           
            return list;
        }



        
        private void btnCreate_Click(object sender, EventArgs e)
        {
            if (dgvLearningSource.DataSource == null)
            {
                MessageBox.Show("Please load some data first.");
                return;
            }

            
            dgvLearningSource.EndEdit();
            /////////////////////////////
             



            ////////////////////////////////
            

           
            double[,] table = (dgvLearningSource.DataSource as DataTable).ToMatrix(out columnNames);

            
                
            double[][] inputs = table.GetColumns(UseParams()).ToArray();
               
            int[] outputs = table.GetColumn(columnNames.Length-1).ToInt32();


           
            svm = new SupportVectorMachine(inputs: columnNames.Length-1);

            
            var smo = new ProbabilisticCoordinateDescent(svm, inputs, outputs)
            {
                
                Complexity = (double)numC.Value,
                Tolerance = (double)numT.Value,
                PositiveWeight = (double)numPositiveWeight.Value,
                NegativeWeight = (double)numNegativeWeight.Value,
            };


            try
            {
              
                double error = smo.Run();

                lbStatus.Text = "Training complete!";
            }
            catch (ConvergenceException)
            {
                lbStatus.Text = "Convergence could not be attained. " +
                    "The learned machine might still be usable.";
            }


           
            double[] weights = svm.Weights.Abs();

            string[] featureNames = columnNames.RemoveAt(columnNames.Length - 1);
            dgvSupportVectors.DataSource = new ArrayDataView(weights, featureNames);

            
        }

        

        


        

      
        private void btnTestingRun_Click(object sender, EventArgs e)
        {
            if (svm == null || dgvTestingSource.DataSource == null)
            {
                MessageBox.Show("Please create a machine first.");
                return;
            }


            
            double[,] table = (dgvTestingSource.DataSource as DataTable).ToMatrix();


            
            double[][] inputs = table.GetColumns(UseParams()).ToArray();

            
             expected = table.GetColumn(columnNames.Length-1).ToInt32();


            int[] output = new int[expected.Length];

          
            for (int i = 0; i < expected.Length; i++)
                output[i] = svm.Compute(inputs[i]) > 0.5 ? 1 : -1;



           
            ConfusionMatrix confusionMatrix = new ConfusionMatrix(output, expected, 1, -1);
            dgvPerformance.DataSource = new[] { confusionMatrix };


            
        }


        private void btnEstimateC_Click(object sender, EventArgs e)
        {
            DataTable source = dgvLearningSource.DataSource as DataTable;

            
            double[,] sourceMatrix = source.ToMatrix(out columnNames);

            
            double[][] inputs = sourceMatrix.GetColumns(UseParams()).ToArray();

           
          double c = SequentialMinimalOptimization.EstimateComplexity(inputs);

            numC.Value = (decimal)c;
        }


        private void MenuFileOpen_Click(object sender, EventArgs e)
        {
            if (openFileDialog.ShowDialog(this) == DialogResult.OK)
            {
                string filename = openFileDialog.FileName;
                string extension = Path.GetExtension(filename);
               
                var filedata = new StreamReader(File.OpenRead(filename)).ReadToEnd();
                if (filedata != null)
                {
                    DataTable tableSource = new DataTable("s");
                    String[] lines = filedata.Split(new char[] { '\n' });
                    double[][] dummyMatrix = new double[lines.Length][];
                    for (int i = 0; i < lines.Length; i++)
                    {
                        String[] columns = lines[i].Split(',');

                        dummyMatrix[i] = new double[columns.Length];
                        if (i == 0)
                        {
                            for (int j = 0; j < columns.Length; j++)
                            {
                                tableSource.Columns.Add(""+j+"");
                            }
                        }

                        if (columns.Length > 1)
                        {
                            DataRow newRow = tableSource.NewRow();

                            for (int j = 0; j < columns.Length; j++)
                            {

                                double val = double.Parse(columns[j]);
                                dummyMatrix[i][j] = val;
                                newRow[j] = val;

                            }
                            tableSource.Rows.Add(newRow);
                        }
                    }
                    double[,] sourceMatrix = new double[dummyMatrix.Count(),dummyMatrix[0].Count()];
                    for(int i = 0;i<dummyMatrix.Count();i++){
                        for(int j=0;j<dummyMatrix[i].Count();j++)
                        {
                            sourceMatrix[i,j]= dummyMatrix[i][j];
                        }
                    }

                    
                    if (sourceMatrix.GetLength(1) == 2)
                    {
                        MessageBox.Show("Missing class column.");
                    }
                    else
                    {
                        this.dgvLearningSource.DataSource = tableSource;
                        this.dgvTestingSource.DataSource = tableSource.Copy();


                       
                    }
                }
            }

            lbStatus.Text = "Switch to the Machine Creation tab to create a learning machine!";
        }





        

        private void toolStripMenuItem5_Click(object sender, EventArgs e)
        {
            Close();
        }

        private void toolStripMenuItem7_Click(object sender, EventArgs e)
        {
           
        }

        private void menuStrip1_ItemClicked(object sender, ToolStripItemClickedEventArgs e)
        {

        }

        private void statusStrip1_ItemClicked(object sender, ToolStripItemClickedEventArgs e)
        {

        }

        private void tabPage5_Click(object sender, EventArgs e)
        {

        }

        

        private void button2_Click_1(object sender, EventArgs e)
        {
            List<double[]> features;

            List<double[]> HaralickFeatures;
            
               i = 0;
               pictureBox1.Image = AForge.Imaging.Image.FromFile(alist[0].ToString());
               do
               {
                   i++;

                   pictureBox1.Image = AForge.Imaging.Image.FromFile(alist[times].ToString());
                   Bitmap OriginalImage = (Bitmap)pictureBox1.Image;
                   
                   Thresholded = (Bitmap)pictureBox1.Image;
                   pictureBox1.Image = Thresholded;

                   HistogramsOfOrientedGradients Hist = new HistogramsOfOrientedGradients(9,3,70);
                   features = Hist.ProcessImage(OriginalImage);

                   lengthOfFeature = features.Count;
                   string path = @"C:\\Users\\pc\\Desktop\\HOG.txt";

                   Haralick h = new Haralick();
                   HaralickFeatures = h.ProcessImage(OriginalImage);

                   using (var file = File.AppendText("C:\\Users\\pc\\Desktop\\HandednessTesting.csv"))
                   {
                       foreach (var arr in features)
                       {
                          
                           file.Write(Math.Round(arr[0], 4).ToString());
                           for (int j = 0; j < arr.Length; j++)
                           {
                               file.Write(',');
                               file.Write(Math.Round(arr[j], 4).ToString());

                           }
                          
                       }
                       
                       file.Write(',');

                       foreach (var arr1 in HaralickFeatures)
                       {
                           file.Write(Math.Round(arr1[0], 4).ToString());
                           for (int j = 1; j < arr1.Length; j++)
                           {
                               file.Write(',');
                               file.Write(Math.Round(arr1[j], 4).ToString());
                           }
                           file.WriteLine();

                       }
                       
                   }
                   
                   times++;
                
                   
                   

               } while (times < 22);
              
            }


        private void button3_Click_1(object sender, EventArgs e)
        {
            string file = "";
            OpenFileDialog openFileDialog1 = new OpenFileDialog();
            DialogResult result = openFileDialog1.ShowDialog(); 
            




             

            try
            {
                int process=0;

                if (result == DialogResult.OK) 
                {
                    file = openFileDialog1.FileName;
                    pictureBox1.ImageLocation = @file;
                    pictureBox1.SizeMode = PictureBoxSizeMode.StretchImage;
                }
                
                DirectoryInfo inputDir = new DirectoryInfo("F:\\Final Year Project Data\\FYPMAchineDATA\\HandednessData\\HandednessValidationData");
                foreach (FileInfo eachfile in inputDir.GetFiles())
               
                {
                    FileInfo file1 = null;
                    file1 = eachfile;

                    if (file1.Extension == ".PNG" || file1.Extension == ".jpg" || file1.Extension == ".bmp" || file1.Extension == ".tif")
                    {
                        alist.Add(file1.FullName); 
                        filelength = filelength + 1;
                    }
                }
                alist.Sort();
               
                
                
            catch (Exception ex)
            {
                MessageBox.Show("Error: Could not read file from disk. Original error: " + ex.Message);
            }
          }
        

        private void button4_Click_1(object sender, EventArgs e)
        {
            if (i - 1 < filelength && i>0)
            {
                pictureBox1.Image = AForge.Imaging.Image.FromFile(alist[i - 1].ToString());
                i = i - 1;
            }
        }

        private void button5_Click_1(object sender, EventArgs e)
        {
            if (i + 1 >= 0 && i <= alist.Count - 2)
            {
                pictureBox1.Image = AForge.Imaging.Image.FromFile(alist[i + 1].ToString());
                i = i + 1;
            }
        }

        private void button6_Click(object sender, EventArgs e)
        {
            this.Close();
        }

        private void dgvLearningSource_CellContentClick(object sender, DataGridViewCellEventArgs e)
        {

        }

        private void button1_Click(object sender, EventArgs e)
        {
            OpenFileDialog openFileDialog1 = new OpenFileDialog();
            DialogResult result = openFileDialog1.ShowDialog();
            try
            {
                if (result == DialogResult.OK) 
                {
                    while (itteration < 12)
                    {
                       
                        pictureBox1.Image = AForge.Imaging.Image.FromFile(alist[itteration].ToString());
                        System.Drawing.Image image = pictureBox1.Image;
                        image = ResizeImage(image, this.Size);

                        itteration++;
                        image.Save("F:\\Final Year Project Data\\Data\\New folder (2)\\" + itteration + ".jpg", System.Drawing.Imaging.ImageFormat.Bmp);
                       
                        
                    }
                }
            }

            catch (Exception ex)
            {
                MessageBox.Show("Error: Could not read file from disk. Original error: " + ex.Message);
            }

        }

        private Bitmap BitmapFromFile(string p)
        {
            throw new NotImplementedException();
        }

        private void toolStripMenuItem1_Click(object sender, EventArgs e)
        {

        }

        private void numC_ValueChanged(object sender, EventArgs e)
        {

        }

        private void dgvPerformance_CellContentClick(object sender, DataGridViewCellEventArgs e)
        {

        }

        private void graphInput_Load(object sender, EventArgs e)
        {

        }

        private void groupBox15_Enter(object sender, EventArgs e)
        {

        }

        private void groupBox3_Enter(object sender, EventArgs e)
        {

        }

        private void groupBox1_Enter(object sender, EventArgs e)
        {

        }

        private void button7_Click(object sender, EventArgs e)
        {
            for (int j = 0; j < expected.Length; j++)
            {
                Gender = expected[j];
            }
            if (Gender == 1)
            {
                MessageBox.Show("Male 0r 1");
            }
            else
            {
                MessageBox.Show("Female Or -1");
            }
            
        }

        private void label2_Click(object sender, EventArgs e)
        {

        }

        
        }
    }

