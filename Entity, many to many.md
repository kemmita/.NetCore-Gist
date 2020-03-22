1. We will have a situation where many teachers can have many students.
```cs
    public class Teacher
    {
        [Key]
        public int Id { get; set; }
        public string Name { get; set; }

        //Foreign Key
        public IList<TeacherStudent>  TeacherStudents { get; set; }
    }
```
```cs
    public class Student
    {
        [Key]
        public int Id { get; set; }
        public string Name { get; set; }

        //Foreign Key
        public IList<TeacherStudent> TeacherStudents { get; set; }
    }
```
```cs
   public class TeacherStudent
    {
        [Key]
        public int TeacherId { get; set; }
        public Teacher Teacher { get; set; }

        [Key]
        public int StudentId { get; set; }
        public Student Student { get; set; }
    }
```
2. Below is your application context class
```cs
public class KemmitContext : DbContext
{

    public KemmitContext() : base("name=KemmitContext")
    {
    }

    public DbSet<Student> Students { get; set; }
    public DbSet<Teacher> Teachers { get; set; }
    public DbSet<TeacherStudent> TeacherStudents { get; set; }

    protected override void OnModelCreating(DbModelBuilder dbModelBuilder)
    {
        dbModelBuilder.Entity<TeacherStudent>().HasKey(ts => new { ts.StudentId, ts.TeacherId});
        dbModelBuilder.Entity<TeacherStudent>().HasRequired(t => t.Teacher)
            .WithMany(ts => ts.TeacherStudents)
            .HasForeignKey(t => t.TeacherId);
        dbModelBuilder.Entity<TeacherStudent>().HasRequired(s => s.Student)
            .WithMany(ts => ts.TeacherStudents)
            .HasForeignKey(s => s.StudentId);
        base.OnModelCreating(dbModelBuilder);
    }

}
```
3. Below is how you would insert/write to a many to many table. In this scenario we have a teacher id and a new student is beign assigned to that teacher.
```cs
        URL: http://localhost:53741/api/students?teacherId=1
        
        [ResponseType(typeof(Student))]
        public IHttpActionResult PostStudent([FromBody]Student student, [FromUri] int teacherId)
        {
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            var teacher = db.Teachers.FirstOrDefault(t => t.Id == teacherId);
            var teacherStudent = new TeacherStudent
            {
                Student = student,
                Teacher = teacher
            };
            db.TeacherStudents.Add(teacherStudent);
            db.Students.Add(student);

            db.SaveChanges();

            return CreatedAtRoute("DefaultApi", new { id = student.Id }, student);
        }
```
4. Below we fetch all of the students associated with a teacher id supplied
```cs
        public IList<Student> GetStudentsOfTeacher(int teacherId)
        {
            return _db.TeacherStudents.Where(ts => ts.TeacherId == teacherId).Select(s => s.Student).ToList();
        }
```
